# Can you use websockets in ServiceNow?

tldr;
Yes! But be advised that we are in undocumented waters, APIs are subject to change, you are all alone here, etc.

This will contain 3 parts - Introduction: Finding the socket in Angular.js, Implementing your own web socket api, and How Graphql uses the Socket in Agent Workspace

I will assume that you know what web sockets are and why/when to use them. The goal is to see how you can use Websockets in _vanilla.js_ anywhere that is needed. Let's look at how you can leverage them in your project.

## Finding the socket - spUtil.recordWatch

First stop on the web socket train is the Service Portal, where we find the only supported mechanism (emphasis on _supported_, as the platform itself uses it elsewhere) for a proper websocket in ServiceNow. AND... we find it in Angular.js of all places.

Indeed, it was the Service Portal that got me thinking, "How does this thing work anyway?"

The `recordWatch "watches for updates to a table or filter and returns the value from the callback function."` Hmm... Let's look at [the example from the docs](https://docs.servicenow.com/bundle/orlando-application-development/page/app-store/dev_portal/API_reference/spUtil/concept/spUtilAPI.html) -

```
 spUtil.recordWatch($scope, "incident", "active=true", function(response) {
    // Returns the data inserted or updated on the table
    console.log(response.data);
    });
}
```

In some way, giving this util a target table + filter allows you to subscribe to server sent events in the UI and register a callback to update Angular.js. Using our handy `CMD-Shift-F` (with anonymous search enabled on scripts) in Chome DevTools, we can look at the implementation.

```
recordWatch: function($scope, table, filter, callback) {
    var watcherChannel = snRecordWatcher.initChannel(table, filter || 'sys_id!=-1');
    var subscribe = callback || function(){ $scope.server.update() };
    watcherChannel.subscribe(subscribe);
    $scope.$on('$destroy', function() {
    watcherChannel.unsubscribe();
    });
},
```

Now we are getting somewhere. This is the key line for our purposes

```
var watcherChannel = snRecordWatcher.initChannel(table, filter || 'sys_id!=-1');
```

ServiceNow has a separate API, also in Angular.js, which builds a channel and exposes the ability to subscribe/unsubscribe. Moving a bit further up the chain, we check out the `initChannel` implementation-

```
function initChannel(table, filter) {
    if (isBlockedTable(table)) {
      $log.log("Blocked from watching", table);
      return null;
    }
    if (diagnosticLog) log(">>> init " + table + "?" + filter);
    watcherChannel = amb.getChannelRW(table, filter);
    watcherChannel.subscribe(onMessage);
    amb.connect();
    return watcherChannel;
}
```

And now we find what we are looking for. `amb`, that old friend that fills up your console with log messages on the platform. The old friend that will completely freeze your instance if you have a breakpoint on a page, including and not limited to locking up your Rest API.

With a little more digging, we discover that `amb` is a shiny `window` object that has _nothing to do with Angular.js_. That's right, Angular.js, ya aint gonna need it. Try it out - open the console and enter `window.amb`. Everything we need is right there.

So, we know that the socket was out there, somewhere, but with a bit of investigative work, Angular.js has shown us the way. Stay tuned for the next installment, where we unpack the Angular implemenation to create our own web socket api without any reference to Angular.js.
