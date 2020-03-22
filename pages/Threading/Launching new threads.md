# Parallelizing ServiceNow Computation

This article reviews the variety of techniques that can be used to run asynchronous logic in ServiceNow. This article assumes that the reader needs to execute long-running logic *eventually*, meaning not in real-time, not on a delayed schedule, but rather in near-real-time. 

## Async Business Rule

The most commonly understood way to process logic asynchronously, the Async BR, is a fairly well understood tool to execute some long-running logic when you don't want your database transactions to take too long (which can result in a poor UX)

You should use an Async BR when you have long running logic that needs to run in response to a database transaction (EG: Fetch latitude/longitude from an external API when an address is entered or changed). 

The downside of an Async BR is that it is highly coupled with the table you define the business rule on. How do we achieve async behavior independently of a database transaction on a particular table? 

### How to use it
Create a BR and trigger it on Async. For more information, read [here](https://docs.servicenow.com/bundle/orlando-application-development/page/script/business-rules/reference/r_HowBusinessRulesWork.html).

## Schedule Worker

Schedule Workers offer a way to dynamically launch a new thread. In fact, Async BR's actually use the same system to launch an asynchronous processing thread. The difference with this method however is that any arbitrary logic can launch a Schedule Worker without requiring that a record is updated, such as a call to a Scripted REST API. In this way, we can achieve asynchronous logic that is decoupled from a database transaction.

> **Note**: The default behavior of a Schedule Worker is to ... *spoiler alert*, run on a schedule. However, in this example we are running a worker on demand, which can save us from wasting system resources on executing a job on a schedule and instead executing it when we need it.

For very simple use cases that need to be de-coupled from a database transaction, Schedule Workers should be the first tool to reach for, since they are 1) [Documented from ServiceNow](https://developer.servicenow.com/dev.do#!/reference/api/orlando/server/r_SGSYS-executeNow_GR) and 2) Cancellation Safe (more on this later)

Some downsides to Schedule Workers are that they do not allow for inputs, therefore, there is no ability to make your code run differently based on data, unless you were to build your own queue system. 

### How to use it
1. Create a Script Scheduled Job (Leave it as `inactive`) -- write your logic here
2. From a script where you want to launch a new thread, include the following snippet:
```javascript
var job = new GlideRecord("sysauto_script");
job.get("{sys_id of scheduled job here}");

gs.executeNow(job);
```
That's it! You will notice that the execution will not halt your current thread, and will run your scheduled job as soon as the system is able. In most cases, that will mean immediately, however if the ServiceNow instance's scheduler is under heavy load, processing of the job may be delayed. 

Some use cases where a schedule worker is appropriate would be to run some fix logic after an event, such as a metric update. Speaking of events...

## Events
By now, you might be thinking, *why did we jump from Async BR to Scheduled Workers? Won't events allow me to run things asynchronously as well without tying myself to a database transaction?*

While events may seem like the solution to running arbitrary logic asynchronously, there are a few considerations to keep in mind when responding to events. The only way to respond to an event is to use a Script Action. However, what the ServiceNow docs don't do a good job of explaining, is that ServiceNow's event queue scheduler job will pick up multiple events in a single thread, and run through those events' Script Actions in a single thread. 

This design means that developers *should* keep Script Action processing to a minimum.  

Let's take the event scheduler's default behavior of picking up 500 events per thread. (Actual number may vary per instance). Assume each of those 500 events runs a Script Action that runs for exactly `one` second. One Script Action running for one second seems innocuous enough -- until you realize that the scheduler that runs those will run each Script Action in series -- thus causing the scheduler to have to run for **at least** `500` seconds!

Now think about the implications -- not only are you at the mercy of assuming the other developers in your ServiceNow instance are courteous enough to know this rule (keep Script Action processing short), **you** should also adhere to this rule! But keeping in mind the assumptions of this article, we need to execute long-running tasks! 

Therefore, ServiceNow's Script Action system should not be the tool we reach for in this scenario. 

So, what **should** you use Script Actions and Events for? Events should be used to send a signal out to the system that *something* happened. For example, if I wanted developers to be able to respond to the fact that a Change Request has closed, but I don't want them to write business rules on my Change Request table -- I could fire an event to notify that a Change Request has closed, along with the necessary information. Then, other developers could run extremely short-running logic to respond to this event, in a Script Action. 

> **Note**: There is no `How to use it` section here because Events should not be used for long-running processing in the background. Period.

By now, the next question you might be asking is -- OK, so what if I have to run long running logic, in near-real-time, and I need to pass inputs to the logic? Don't tell ICE, because here is where things get a bit more... undocumented. 

## Hierarchical Progress Workers

Hierarchical Progress Workers, or HPW for short, are an extremely flexible way of running asynchronous logic when you need to carefully control the inputs. Dynamic inputs can be passed in memory rather than storing them in a queue table (which would be necessary to get different results on each execution using the Schedule Worker method).

The downfall of HPW as compared to Schedule Workers, is that the system reserves the right to cancel these at any time, which means it is up to the developer to ensure that requests made to HPWs are robust enough to handle being cancelled. 

### How to use it

1. Put your desired async logic into a Script Include method
2. From a script where you want to launch a new thread, include the following snippet:
```javascript
var worker = new GlideScriptedHierarchicalWorker();
worker.setProgressName('Worker Name Here');
worker.setBackground(true);
worker.setScriptIncludeName('scope.{name_of_script_include}');
worker.putConstructorArg(
    {name_of_constructor_argument}, 
    {value_of_constructor_argument});
worker.setScriptIncludeMethod('{name_of_method}');
worker.putMethodArg(
    {name_of_method_argument},
    {value_of_method_argument});
worker.start();
```
