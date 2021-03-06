# Durable Functions Overview

## What are Durable Functions?
"Durable Functions" is an extension of Azure Functions and Azure WebJobs that allows writing *long-running*, *stateful* function orchestrations in code in a serverless environment.

A new type of function called the *orchestrator function* allows you to do several new things that differentiates it from an ordinary, stateless function:
* They are stateful workflows **authored in code**. No JSON schemas or designers.
* They can *synchronously* and *asynchronously* **call other functions** and **save output to local variables**.
* They **automatically checkpoint** their progress whenever the function awaits so that local state is never lost if the process recycles or the VM reboots.

> [!NOTE]
> This is an advanced feature of Azure Functions and may not be useful for all applications. The rest of this article assumes the reader has a strong familiarity with existing Azure Functions concepts and the challenges involved in serverless application development.

## Patterns
The primary use case for Durable Functions is *simplifying complex, stateful coordination problems* in serverless applications. To understand whether your application can benefit from Durable Functions, it's useful to look at a few example coordination patterns, which we'll walk through below.

### Pattern #1: Function Chaining
Function chaining refers to the common pattern executing a sequence of discrete functions in a particular order. Often the output of one function needs to be applied to the input of another function.

<img src="~/images/function-chaining.png"/>

Durable Functions allows implementing this pattern as a simple, easy to understand C# function snippet.

```cs
public static async Task<object> Run(DurableOrchestrationContext ctx)
{
    try
    {
        var x = await ctx.CallFunctionAsync<object>("F1");
        var y = await ctx.CallFunctionAsync<object>("F2", x);
        var z = await ctx.CallFunctionAsync<object>("F3", y);
        return  await ctx.CallFunctionAsync<object>("F4", z);
    }
    catch (Exception)
    {
        // error handling/compensation goes here
    }
}
```

This code above is concise and easily understood by any developer. Control flow is implemented using normal imperitive coding constructs - i.e. code executes top-down and can involve existing language control flow semantics, like conditionals, loops, try/catch/finally, etc., and local variables are used to capture and flow state. If more detailed compensation logic is required, it can be done so naturally with additional try/catch logic.

The built-in `ctx` parameter provides methods for invoking other functions by name, passing parameters, and returning function output. Each time the code calls `await`, the orchestrator function *checkpoints* the progress of the current function instance. If the process or VM recycles midway through the execution, the function instance will resume from the previous `await` call, much like it would if we were using queues in between each function invocation. More on this later.

### Pattern #2: Fan-out, Fan-in
A more interesting pattern enabled by the Durable Functions extension is fan-out, fan-in.

<img src="~/images/fan-out-fan-in.png"/>

With normal functions, fanning out can be done very easily by having the function output multiple messages to a queue using an output binding. However, fanning back in is much more challenging because work needs to be done to coordinate and orchestrate the data processing.

The Durable Functions extension is optimized to handle this kind of pattern with a relatively simple code snippet.

```cs
public static async Task Run(DurableOrchestrationContext ctx)
{
    var parallelTasks = new List<Task<int>>();
 
    // get a list of N work items to process in parallel
    object[] workBatch = await ctx.CallFunctionAsync<object[]>("F1");
    for (int i = 0; i < workBatch.Length; i++)
    {
        Task<int> task = ctx.CallFunctionAsync<int>("F2", workBatch[i]);
        parallelTasks.Add(task);
    }
 
    await Task.WhenAll(parallelTasks);
 
    // aggregate all N outputs and send result to F3
    int sum = parallelTasks.Sum(t => t.Result);
    await ctx.CallFunctionAsync("F3", sum);
}
```

As demonstrated by the code, the fan-out work is tracked using a dynamic list of tasks. Then using the existing .NET `Task.WhenAll` API, a continuation can be scheduled which aggregates the outputs from the dynamic task list and passes it onto a third function. The automatic checkpointing that happens at the `await` call on `Task.WhenAll` ensures that any crash or reboot midway through does not require a restart of any already completed tasks.

### Pattern #3: Async HTTP APIs
The third pattern is all about the problem of coordinating the state of long-running operations with external clients. A common way to implement this pattern is by having the long-running action triggered by an HTTP call, and then redirecting the client to a status endpoint that they can poll to learn when the operation completes.

<img src="~/images/async-http-api.png"/>

Durable functions makes implementing these kinds of Async HTTP APIs very simple by providing a set of built-in HTTP APIs for interacting with long-running function executions. We show in the samples a simple REST commands can be used to start new orchestrator function instances. Once an instance is started, the durable extension exposes web hook HTTP APIs that can be used to query their status.

<pre>
> curl -X POST https://myfunc.azurewebsites.net/orchestrators/DoWork -H "Content-Length: 0" -i
HTTP/1.1 202 Accepted
Content-Type: application/json
Location: https://myfunc.azurewebsites.net/admin/extensions/DurableTaskConfiguration/b79baf67f717453ca9e86c5da21e03ec

{"id":"b79baf67f717453ca9e86c5da21e03ec", ...}

> curl https://myfunc.azurewebsites.net/admin/extensions/DurableTaskConfiguration/b79baf67f717453ca9e86c5da21e03ec -i
HTTP/1.1 202 Accepted
Content-Type: application/json
Location: https://myfunc.azurewebsites.net/admin/extensions/DurableTaskConfiguration/b79baf67f717453ca9e86c5da21e03ec

{"runtimeStatus":"Running","lastUpdatedTime":"2017-03-16T21:20:47Z", ...}

> curl https://myfunc.azurewebsites.net/admin/extensions/DurableTaskConfiguration/b79baf67f717453ca9e86c5da21e03ec -i
HTTP/1.1 200 OK
Content-Length: 175
Content-Type: application/json

{"runtimeStatus":"Completed","lastUpdatedTime":"2017-03-16T21:20:57Z", ...}
</pre>

Because all the state is managed by the Durable Functions runtime, it's not necessary for you to implement your own status tracking mechanism.

It's important to point out that even though the Durable Functions extension has built-in web hooks for managing long-running orchestrations, it's entirely possible for you to implement this pattern yourself using your own function triggers (e.g. HTTP, queue, Event Hub, etc.) and the `orchestrationClient` binding.

```cs
// HTTP-triggered function to start a new orchestrator function instance.
public static async Task<HttpResponseMessage> Run(
    HttpRequestMessage req,
    DurableOrchestrationClient starter,
    string functionName,
    TraceWriter log)
{
    // Function name comes from the request URL.
    // Function input comes from the request content.
    dynamic eventData = await req.Content.ReadAsAsync<object>();
    string instanceId = await starter.StartNewAsync(functionName, eventData);
    
    log.Info($"Started orchestration with ID = '{instanceId}'.");
    
    return starter.CreateCheckStatusResponse(req, instanceId);
}
```

The <xref:Microsoft.Azure.WebJobs.DurableOrchestrationClient> `starter` parameter is a value from the `orchestrationClient` output binding, which is part of the Durable Functions extension. It provides methods for starting, sending events to, terminating, and querying for new or existing orchestrator function instances. In the above example, an HTTP triggered-function takes in a `functionName` value from the incoming URL and passes that value to <xref:Microsoft.Azure.WebJobs.DurableOrchestrationClient.StartNewAsync*>. This binding API then returns response that contains a `Location` header and additional information about the instance that can later be used to look up the status of or terminate the started instance.

### Pattern #4: Lightweight Actors
The [Actor model](https://en.wikipedia.org/wiki/Actor_model) is a pattern that is becoming more common in distributed computing, particularly in the cloud.

<img src="~/images/actor.png"/>

Because ordinary functions in Azure Functions are stateless, they are not well suited to implement this kind of stateful pattern. With the Durable Functions extension it is relatively trivial to implement stateful actors. Below is an example orchestrator function which implements a counter.

```cs
public static async Task Run(DurableOrchestrationContext ctx)
{
    int counterState = ctx.GetInput<int>();

    string operation = await ctx.WaitForExternalEvent<string>("operation");
    if (operation == "incr")
    {
        counterState++;
    }
    else if (operation == "decr")
    {
        counterState--;
    }

    ctx.ContinueAsNew(counterState);
}
```
The above function is what you might describe as an "eternal orchestration" - i.e. one that starts and never ends. It starts with some input value `counterState`, waits indefinitely for a message called `operation`, performs some logic to update its local state, "restarts" itself by calling `ctx.ContinueAsNew`, and then awaits indefinitely for the next operation.

### Pattern #5: Human Interaction and Timeouts
Many types of process automation often need to involve some kind of human interaction. The tricky thing about involving humans in an automated process is that people are not always as highly available and responsive as cloud services. These automated processes must account for this, and often do so using timeouts and compensation.

One example of a business process which involves human interaction is an approval process. One could imagine a system where a manager must approve an expense report that exceeds a certain amount. If the manager does not approve within 72 hours (maybe they went on vacation), then an escalation processes kicks in to get the approval from someone else (perhaps the manager's manager).

<img src="~/images/approval.png"/>

This kind of pattern can be implemented using an orchestrator function to coordinate a "durable timeout" with an "external event" notification generated by some human interaction.

```cs
public static async Task Run(DurableOrchestrationContext ctx)
{
    await ctx.CallFunctionAsync("RequestApproval");
    using (var timeoutCts = new CancellationTokenSource())
    {
        DateTime dueTime = ctx.CurrentUtcDateTime.AddHours(72);
        Task durableTimeout = ctx.CreateTimer(dueTime, timeoutCts.Token);

        Task<bool> approvalEvent = ctx.WaitForExternalEvent<bool>("ApprovalEvent");
        if (approvalEvent == await Task.WhenAny(approvalEvent, durableTimeout))
        {
            timeoutCts.Cancel();
            await ctx.CallFunctionAsync("HandleApproval", approvalEvent.Result);
        }
        else
        {
            await ctx.CallFunctionAsync("Escalate");
        }
    }
}
```
The important thing to notice here is that the timeout was implemented using `ctx.CreateTimer`, `ctx.WaitForExternalEvent`, and `Task.WhenAny` to synchronize the notification and the timer expiration. Depending on which event occurs first, the orchestrator function takes the appropriate action (escalation or completing the approval process).

## The Technology
Behind the scenes the Durable Functions extension is built on top of the [Durable Task Framework](https://github.com/Azure/durabletask), an open source library on GitHub. Much like how Azure Functions is the serverless evolution of Azure WebJobs, Durable Functions is the serverless evolution of the Durable Task Framework. It is used heavily within Microsoft and outside as well to automate mission-critical processes and is a natural fit to leverage within the serverless Azure Functions environment.

### Event Sourcing, Checkpointing, and Orchestrator Replay
Orchestrator functions reliably maintain their execution state using a cloud design pattern known as [Event Sourcing](https://docs.microsoft.com/en-us/azure/architecture/patterns/event-sourcing). Instead of directly storing the *current* state of an orchestration, the durable extension uses an append-only store to record the *full series of actions* taken by the function orchestration. This has many benefits, including improving performance, scalability, and responsiveness compared to "dumping" the full runtime state. Other benefits include providing eventual consistency for transactional data and maintaining full audit trails/history, which itself can enable reliable compensating actions.

The use of Event Sourcing by this extension is completely transparent. Under the covers, the `await` operator in an orchestrator function yields control of the thread back to the Durable Task Framework dispatcher. The dispatcher then commits any new actions that the orchestrator function scheduled (such as calling one or more child functions or scheduling a durable timer) to storage. This transparent commit action appends to the *execution history* of the orchestration instance to durable storage and subsequently add messages to a queue to schedule the actual work. At this point, the orchestrator function can be unloaded from memory and billing can be stopped (if using the Azure Functions Consumption Plan) until there is more work to do, at which point its state can be reconstructed.

Once an orchestration function is given more work to do (e.g. a response message is received or a durable timer expires), the orchestrator will wake up again and **re-execute the entire function from the start** in order to rebuild the local state. If during this replay the code tries to call a function (or do any other async work), the Durable Task Framework will consult with the *execution history* of the current orchestration. If it finds that the activity function has already executed and yielded some result, it will replay that function's result immediately and the orchestrator code will continue running. This will continue happening until the function code gets to a point where either it is finished or it has scheduled new async work.

### Orchestrator Code Constraints
With this replay behavior in mind, there are a very important set of constraints on the type of code that can be written in an orchestrator function:
* Orchestrator code **must be deterministic** since it is going to be replayed multiple times. This means there cannot be any direct calls to get the current date/time, get random numbers, generate random GUIDs, or call into remote endpoints.
* If orchestrator code needs to get the current date/time, it should use the <xref:Microsoft.Azure.WebJobs.DurableOrchestrationContext.CurrentUtcDateTime> API, which is safe for replay.
* Non-deterministic operations need to be done in activity functions. This includes any interaction with other input or output bindings. This ensures any non-deterministic values will be generated once on the first execution and saved into the execution history. Subsequent executions will then use the saved value automatically.
* Orchestrator code should be **non-blocking** - i.e. no `Thread.Sleep` or equivalent APIs. If an orchestrator needs to delay for a period of time, it should use the <xref:Microsoft.Azure.WebJobs.DurableOrchestrationContext.CreateTimer*> API.
* Orchestrator code must never initiate any async operation outside of the operations exposed by <xref:Microsoft.Azure.WebJobs.DurableOrchestrationContext> - e.g. no `Task.Delay` or `HttpClient.SendAsync`. The Durable Task Framework executes orchestrator code on a single thread and cannot interact with any other threads which could be scheduled by other async APIs.
* Because the Durable Task Framework saves execution history as the orchestration function progresses, **infinite loops should be avoided** to ensure orchestrator instances do not run out of memory. Instead, APIs such as <xref:Microsoft.Azure.WebJobs.DurableOrchestrationContext.ContinueAsNew*> should be used to restart the function execution and discard previous execution history.

While these constraints may seem daunting at first, in practice they are quite easy to follow. Where possible, the Durable Task Framework will attempt to detect violations of the above rules and throw a `NonDeterministicOrchestrationException`. However, this detection behavior is best-effort and orchestrator code should never depend on it.

> [!NOTE]
> Note that all the rules mentioned above only apply to functions triggered by the `orchestrationTrigger` binding. Activity functions triggered by the `activityTrigger` binding and functions which use the `orchestrationClient` binding have no such limitations.

## Language Support
Currently C# is the only supported language for Durable Functions. This includes authoring orchestrator functions and activity functions. In the future, we will add support for all languages supported by Azure Functions.

> [!NOTE]
> It's actually possible to write *activity functions* in any supported languages, including JavaScript (node.js). However, return values in these languages are currently not working, which limits the usefulness of using non-C# languages. If you are not using return values, then using languages other than C# are expected to work without issue.

## Monitoring and Diagnostics
The Durable Functions host extension automatically emits semi-structured logs as it executes orchestrator and activity functions. These logs live alongside the application logs and can be used to monitor the behavior of your orchestrations.

Because of the replay behavior of the Durable Task Framework dispatcher, you can expect to see redundant log entries for replayed actions. This can be useful to understand the replay behavior of the core engine. If you would like to see just the "real-time" logs, you can use `grep` or `findstr` to filter log entries based on the `IsReplay` text.

Example logs (no filtering):

<pre>
2017-05-01T22:47:49.048 [DF] e35a4e6ce5984c81850fc6e1f2573262: Scheduling function 'ProcessWorkBatch', version ''. reason: NewInstance. IsReplay: False.
2017-05-01T22:47:49.481 [DF] e35a4e6ce5984c81850fc6e1f2573262: Starting function 'ProcessWorkBatch', version ''. IsReplay: False. Input: (1 bytes)
2017-05-01T22:47:49.505 [DF] e35a4e6ce5984c81850fc6e1f2573262: Scheduling function 'HelloWorld', version ''. reason: ProcessWorkBatch. IsReplay: False.
2017-05-01T22:47:49.509 [DF] e35a4e6ce5984c81850fc6e1f2573262: Function 'ProcessWorkBatch', version '' awaited. IsReplay: False.
2017-05-01T22:47:49.789 [DF] e35a4e6ce5984c81850fc6e1f2573262: Starting function 'HelloWorld', version ''. IsReplay: False. Input: (9 bytes)
2017-05-01T22:47:51.361 [DF] e35a4e6ce5984c81850fc6e1f2573262: Function 'HelloWorld', version '' completed. ContinuedAsNew: False. IsReplay: False. Output: (null)
2017-05-01T22:47:51.451 [DF] e35a4e6ce5984c81850fc6e1f2573262: Starting function 'ProcessWorkBatch', version ''. IsReplay: True. Input: (1 bytes)
2017-05-01T22:47:51.452 [DF] e35a4e6ce5984c81850fc6e1f2573262: Scheduling function 'HelloWorld', version ''. reason: ProcessWorkBatch. IsReplay: True.
2017-05-01T22:47:51.454 [DF] e35a4e6ce5984c81850fc6e1f2573262: Function 'ProcessWorkBatch', version '' awaited. IsReplay: True.
2017-05-01T22:47:51.462 [DF] e35a4e6ce5984c81850fc6e1f2573262: Function 'ProcessWorkBatch', version '' completed. ContinuedAsNew: False. IsReplay: False. Output: (null)
</pre>

...and with replay logs filtered out (e.g. `findstr /C:"IsReplay: False"`)...

<pre>
2017-05-01T22:47:49.048 [DF] e35a4e6ce5984c81850fc6e1f2573262: Scheduling function 'ProcessWorkBatch', version ''. reason: NewInstance. IsReplay: False.
2017-05-01T22:47:49.481 [DF] e35a4e6ce5984c81850fc6e1f2573262: Starting function 'ProcessWorkBatch', version ''. IsReplay: False. Input: (1 bytes)
2017-05-01T22:47:49.505 [DF] e35a4e6ce5984c81850fc6e1f2573262: Scheduling function 'HelloWorld', version ''. reason: ProcessWorkBatch. IsReplay: False.
2017-05-01T22:47:49.509 [DF] e35a4e6ce5984c81850fc6e1f2573262: Function 'ProcessWorkBatch', version '' awaited. IsReplay: False.
2017-05-01T22:47:49.789 [DF] e35a4e6ce5984c81850fc6e1f2573262: Starting function 'HelloWorld', version ''. IsReplay: False. Input: (9 bytes)
2017-05-01T22:47:51.361 [DF] e35a4e6ce5984c81850fc6e1f2573262: Function 'HelloWorld', version '' completed. ContinuedAsNew: False. IsReplay: False. Output: (null)
2017-05-01T22:47:51.462 [DF] e35a4e6ce5984c81850fc6e1f2573262: Function 'ProcessWorkBatch', version '' completed. ContinuedAsNew: False. IsReplay: False. Output: (null)
</pre>

For the best monitoring and diagnostics experience, it is recommended that you enable [Application Insights integration](https://blogs.msdn.microsoft.com/appserviceteam/2017/04/06/azure-functions-application-insights/). This will allow you to store execution logs for longer periods of time and do more efficient query and analysis.

## Storage and Scalability
The Durable Functions extension uses Azure Storage queues, tables, and blobs to persist execution history state and messages that are used to trigger function execution. The default storage account for the function app can be used, or you can configure a separate storage account to create greater isolation in terms of storage throughput limits. The orchestrator code you write does not need to (and should not) interact with the entities in these storage accounts - the entities are managed directly by the Durable Task Framework as an implementation detail.

Orchestrator functions schedule activity functions and receive their responses via internal queue messages. When running in the Azure Functions Consumption plan, these queues are monitored by the Azure Functions Scale Controller and new compute instances are added as needed. When scaled out to multiple VMs, an orchestrator function may run on one VM while activity functions it calls run on several different VMs. You can find more details on the scale behavior of Durable Functions can be found in the <a href="./topics/perf-and-scale.md">Performance and Scale</a> topic.

Lastly, table storage is used to store the execution history for orchestrator accounts. Whenever an instance rehydrates on a particular VM, it fetches its execution history from table storage so that it can rebuild its local state. One of the convenient things about having the history available in Table storage is that you can take a look and see the history of your orchestrations using tools such as [Microsoft Azure Storage Explorer](https://docs.microsoft.com/en-us/azure/vs-azure-tools-storage-manage-with-storage-explorer).

<img src="~/images/storage-explorer.png"/>

> [!WARNING]
> While it's easy and convenient to see execution history in table storage, you should not take any dependency on this table as the specifics of its usage may change as the Durable Functions extension evolves.
