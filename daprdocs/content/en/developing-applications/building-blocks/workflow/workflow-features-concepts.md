---
type: docs
title: "Features and concepts"
linkTitle: "Features and concepts"
weight: 2000
description: "Learn more about the Dapr Workflow features and concepts"
---

Now that you've learned about the [workflow building block]({{< ref workflow-overview.md >}}) at a high level, let's deep dive into the features and concepts included with the Dapr Workflow engine and SDKs. Dapr Workflow exposes several core features and concepts which are common across all supported languages. 

{{% alert title="Note" color="primary" %}}
For more information on how workflow state is managed, see the [workflow architecture guide]({{< ref workflow-architecture.md >}}).
{{% /alert %}}

## Workflows

Dapr Workflows are functions you write that define a series of steps or tasks to be executed in a particular order. The Dapr Workflow engine takes care of coordinating and managing the execution of the steps, including managing failures and retries. If the app hosting your workflows is scaled out across multiple machines, the workflow engine may also load balance the execution of workflows and their tasks across multiple machines.

There are several different kinds of tasks that a workflow can schedule, including
- [Activities]({{< ref "workflow-features-concepts.md#workflow-activities" >}}) for executing custom logic
- [Durable timers]({{< ref "workflow-features-concepts.md#durable-timers" >}}) for putting the workflow to sleep for arbitrary lengths of time
- [Child workflows]({{< ref "workflow-features-concepts.md#child-workflows" >}}) for breaking larger workflows into smaller pieces
- [External event waiters]({{< ref "workflow-features-concepts.md#external-events" >}}) for blocking workflows until they receive external event signals. These tasks are described in more details in their corresponding sections.

### Workflow identity

Each workflow you define has a type name, and individual executions of a workflow have a unique _instance ID_. Workflow instance IDs can be generated by your app code, which is useful when workflows correspond to business entities like documents or jobs, or can be auto-generated UUIDs. A workflow's instance ID is useful for debugging and also for managing workflows using the [Workflow APIs]({{< ref workflow_api.md >}}).

Only one workflow instance with a given ID can exist at any given time. However, if a workflow instance completes or fails, its ID can be reused by a new workflow instance. Note, however, that the new workflow instance effectively replaces the old one in the configured state store.

### Workflow replay

Dapr Workflows maintain their execution state by using a technique known as [event sourcing](https://learn.microsoft.com/azure/architecture/patterns/event-sourcing). Instead of storing the current state of a workflow as a snapshot, the workflow engine manages an append-only log of history events that describe the various steps that a workflow has taken. When using the workflow SDK, these history events are stored automatically whenever the workflow "awaits" for the result of a scheduled task.

When a workflow "awaits" a scheduled task, it unloads itself from memory until the task completes. Once the task completes, the workflow engine schedules the workflow function to run again. This second workflow function execution is known as a _replay_. 

When a workflow function is replayed, it runs again from the beginning. However, when it encounters a task that already completed, instead of scheduling that task again, the workflow engine:

1. Returns the result of the completed task to the workflow.
1. Continues execution until the next "await" point. 

This "replay" behavior continues until the workflow function completes or fails with an error.

Using this replay technique, a workflow is able to resume execution from any "await" point as if it had never been unloaded from memory. Even the values of local variables from previous runs can be restored without the workflow engine knowing anything about what data they stored. This ability to restore state makes Dapr Workflows _durable_ and _fault tolerant_.

{{% alert title="Note" color="primary" %}}
The workflow replay behavior described here requires that workflow function code be _deterministic_. Deterministic workflow functions take the exact same actions when provided the exact same inputs. [Learn more about the limitations around deterministic workflow code.]({{< ref "workflow-features-concepts.md#workflow-determinism-and-code-constraints" >}})
{{% /alert %}}


### Infinite loops and eternal workflows

As discussed in the [workflow replay]({{< ref "#workflow-replay" >}}) section, workflows maintain a write-only event-sourced history log of all its operations. To avoid runaway resource usage, workflows must limit the number of operations they schedule. For example, ensure your workflow doesn't:

- Use infinite loops in its implementation
- Schedule thousands of tasks.

You can use the following two techniques to write workflows that may need to schedule extreme numbers of tasks:

1. **Use the _continue-as-new_ API**:  
    Each workflow SDK exposes a _continue-as-new_ API that workflows can invoke to restart themselves with a new input and history. The _continue-as-new_ API is especially ideal for implementing "eternal workflows", like monitoring agents, which would otherwise be implemented using a `while (true)`-like construct. Using _continue-as-new_ is a great way to keep the workflow history size small.

1. **Use child workflows**:  
    Each workflow SDK exposes an API for creating child workflows. A child workflow behaves like any other workflow, except that it's scheduled by a parent workflow. Child workflows have:
    - Their own history 
    - The benefit of distributing workflow function execution across multiple machines. 
    
    If a workflow needs to schedule thousands of tasks or more, it's recommended that those tasks be distributed across child workflows so that no single workflow's history size grows too large.

### Updating workflow code

Because workflows are long-running and durable, updating workflow code must be done with extreme care. As discussed in the [workflow determinism]({{< ref "#workflow-determinism-and-code-constraints" >}}) limitation section, workflow code must be deterministic. Updates to workflow code must preserve this determinism if there are any non-completed workflow instances in the system. Otherwise, updates to workflow code can result in runtime failures the next time those workflows execute.

[See known limitations]({{< ref "workflow-features-concepts.md#workflow-determinism-and-code-constraints" >}})

## Workflow activities

Workflow activities are the basic unit of work in a workflow and are the tasks that get orchestrated in the business process. For example, you might create a workflow to process an order. The tasks may involve checking the inventory, charging the customer, and creating a shipment. Each task would be a separate activity. These activities may be executed serially, in parallel, or some combination of both.

Unlike workflows, activities aren't restricted in the type of work you can do in them. Activities are frequently used to make network calls or run CPU intensive operations. An activity can also return data back to the workflow.

The Dapr Workflow engine guarantees that each called activity is executed **at least once** as part of a workflow's execution. Because activities only guarantee at-least-once execution, it's recommended that activity logic be implemented as idempotent whenever possible.

## Child workflows

In addition to activities, workflows can schedule other workflows as _child workflows_. A child workflow has its own instance ID, history, and status that is independent of the parent workflow that started it.

Child workflows have many benefits:

* You can split large workflows into a series of smaller child workflows, making your code more maintainable.
* You can distribute workflow logic across multiple compute nodes concurrently, which is useful if your workflow logic otherwise needs to coordinate a lot of tasks.
* You can reduce memory usage and CPU overhead by keeping the history of parent workflow smaller.

The return value of a child workflow is its output. If a child workflow fails with an exception, then that exception is surfaced to the parent workflow, just like it is when an activity task fails with an exception. Child workflows also support automatic retry policies.

{{% alert title="Note" color="primary" %}}
Because child workflows are independent of their parents, terminating a parent workflow does not affect any child workflows. You must terminate each child workflow independently using its instance ID.
{{% /alert %}}

## Durable timers

Dapr Workflows allow you to schedule reminder-like durable delays for any time range, including minutes, days, or even years. These _durable timers_ can be scheduled by workflows to implement simple delays or to set up ad-hoc timeouts on other async tasks. More specifically, a durable timer can be set to trigger on a particular date or after a specified duration. There are no limits to the maximum duration of durable timers, which are internally backed by internal actor reminders. For example, a workflow that tracks a 30-day free subscription to a service could be implemented using a durable timer that fires 30-days after the workflow is created. Workflows can be safely unloaded from memory while waiting for a durable timer to fire.

{{% alert title="Note" color="primary" %}}
Some APIs in the workflow authoring SDK may internally schedule durable timers to implement internal timeout behavior.
{{% /alert %}}

## External events

Sometimes workflows will need to wait for events that are raised by external systems. For example, an approval workflow may require a human to explicitly approve an order request within an order processing workflow if the total cost exceeds some threshold. Another example is a trivia game orchestration workflow that pauses while waiting for all participants to submit their answers to trivia questions. These mid-execution inputs are referred to as _external events_.

External events have a _name_ and a _payload_ and are delivered to a single workflow instance. Workflows can create "_wait for external event_" tasks that subscribe to external events and _await_ those tasks to block execution until the event is received. The workflow can then read the payload of these events and make decisions about which next steps to take. External events can be processed serially or in parallel. External events can be raised by other workflows or by workflow code.

{{% alert title="Note" color="primary" %}}
The ability to raise external events to workflows is not included in the alpha version of Dapr's workflow API.
{{% /alert %}}

Workflows can also wait for multiple external event signals of the same name, in which case they are dispatched to the corresponding workflow tasks in a first-in, first-out (FIFO) manner. If a workflow receives an external event signal but has not yet created a "wait for external event" task, the event will be saved into the workflow's history and consumed immediately after the workflow requests the event.

## Limitations

### Workflow determinism and code restraints 

To take advantage of the workflow replay technique, your workflow code needs to be deterministic. For your workflow code to be deterministic, you may need to work around some limitations.

#### Workflow functions must call deterministic APIs. 
APIs that generate random numbers, random UUIDs, or the current date are _non-deterministic_. To work around this limitation, you can:
 - Use these APIs in activity functions, or 
 - (Preferred) Use built-in equivalent APIs offered by the SDK. For example, each authoring SDK provides an API for retrieving the current time in a deterministic manner.  

For example, instead of this:

{{< tabs ".NET" >}}

{{% codetab %}}

```csharp
// DON'T DO THIS!
DateTime currentTime = DateTime.UtcNow;
Guid newIdentifier = Guid.NewGuid();
string randomString = GetRandomString();
```

{{% /codetab %}}

{{< /tabs >}}

Do this:

{{< tabs ".NET" >}}

{{% codetab %}}

```csharp
// Do this!!
DateTime currentTime = context.CurrentUtcDateTime;
Guid newIdentifier = context.NewGuid();
string randomString = await context.CallActivityAsync<string>("GetRandomString");
```

{{% /codetab %}}

{{< /tabs >}}


#### Workflow functions must only interact _indirectly_ with external state. 
External data includes any data that isn't stored in the workflow state. Workflows must not interact with global variables, environment variables, the file system, or make network calls. 
    
Instead, workflows should interact with external state _indirectly_ using workflow inputs, activity tasks, and through external event handling.

For example, instead of this:

```csharp
// DON'T DO THIS!
string configuration = Environment.GetEnvironmentVariable("MY_CONFIGURATION")!;
string data = await new HttpClient().GetStringAsync("https://example.com/api/data");
```

Do this:

```csharp
// Do this!!
string configuation = workflowInput.Configuration; // imaginary workflow input argument
string data = await context.CallActivityAsync<string>("MakeHttpCall", "https://example.com/api/data");
```

#### Workflow functions must execute only on the workflow dispatch thread.  
The implementation of each language SDK requires that all workflow function operations operate on the same thread (goroutine, etc.) that the function was scheduled on. Workflow functions must never:
- Schedule background threads, or
- Use APIs that schedule a callback function to run on another thread. 
    
Failure to follow this rule could result in undefined behavior. Any background processing should instead be delegated to activity tasks, which can be scheduled to run serially or concurrently.

For example, instead of this:

```csharp
// DON'T DO THIS!
Task t = Task.Run(() => context.CallActivityAsync("DoSomething"));
await context.CreateTimer(5000).ConfigureAwait(false);
```

Do this:

```csharp
// Do this!!
Task t = context.CallActivityAsync("DoSomething");
await context.CreateTimer(5000).ConfigureAwait(true);
```

### Updating workflow code

Make sure updates you make to the workflow code maintain its determinism. A couple examples of code updates that can break workflow determinism:

- **Changing workflow function signatures**:  
   Changing the name, input, or output of a workflow or activity function is considered a breaking change and must be avoided.  

- **Changing the number or order of workflow tasks**:   
   Changing the number or order of workflow tasks causes a workflow instance's history to no longer match the code and may result in runtime errors or other unexpected behavior.

To work around these constraints:

- Instead of updating existing workflow code, leave the existing workflow code as-is and create new workflow definitions that include the updates. 
- Upstream code that creates workflows should only be updated to create instances of the new workflows. 
- Leave the old code around to ensure that existing workflow instances can continue to run without interruption. If and when it's known that all instances of the old workflow logic have completed, then the old workflow code can be safely deleted.


## Next steps

{{< button text="Workflow patterns >>" page="workflow-patterns.md" >}}

## Related links

- [Try out Dapr Workflow using the quickstart]({{< ref workflow-quickstart.md >}})
- [Workflow overview]({{< ref workflow-overview.md >}})
- [Workflow API reference]({{< ref workflow_api.md >}})
- [Try out the .NET example](https://github.com/dapr/dotnet-sdk/tree/master/examples/Workflow)