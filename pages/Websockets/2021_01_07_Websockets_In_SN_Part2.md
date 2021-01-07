# Can I use Websockets in ServiceNow?  | Implementing your own Websocket API

Yes! This is the 2nd part of our series (out of 3) on how to use Websockets in ServiceNow. You can see [part 1 here - Websockets in ServiceNow recordWatcher](https://engineering.nuvolo.com/can-you-use-websockets-in-servicenow), and part 3 here (whenever it is published - Agent Workspace, Graphql, and the Websocket). 

## TLDR 

To get started right away with a vanilla js Websocket in ServiceNow, you can use [our open-source package SNSocket](https://github.com/nuvolo/sn-socket), which is a thin wrapper over the `amb` api that ServiceNow leverages under the hood. There you can find the latest installation and how-to details

## Websocket Infrastructure in ServiceNow

In order to use a Websocket, the client `"...has to start the WebSocket handshake process by contacting the server and requesting a WebSocket connection"` [source](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API/Writing_WebSocket_servers#the_websocket_handshake). The server, if configured correctly, will respond with an upgraded connection. 

Fortunately for our purposes, the infrastructure is already in place. ServiceNow uses a WebSocket to power its `amb` api. If you navigate to any plain olde ServiceNow backend and open the Network tab, you can see the upgraded connection

![Socket Upgrade](images/snSocketNetworkUpgrade.png)

We can leverage this information to roll our own Websocket wrapper.


## AMB api

As of today, there is [no documented way](https://developer.servicenow.com/dev.do#!/search/latest/All/AMB) to use the amb (Asynchronous Message Bus). However, since we already tracked the api through the `recordWatcher` in part 1 of this series, we can look briefly at the `amb` code to create our own watcher in vanilla js.

The first piece of `amb` to consider is how to load it in a scripting context. It is as simple as loading the script from the ServiceNow instance. Since we are serving this from ServiceNow, there's no security concerns and you can use the relative path. Loading this file will handle all of the initialization and provide a nice and tidy `amb` object on the `window`. At the time of publishing, you can do something like this

```js
<script src="./scripts/glide-amb-client-bundle.min.js" type="text/javascript"></script>
```

Once the library is loaded in our context (UI Page, etc.), we can look at the mechanism for accessing the pipe. `amb` exposes a client, which in turn exposes channel. It is via this channel that one can ride all the way to the DB, listen, and fire a callback when the subscription is notified of a change. Easy-peasy

```js
const callback = (payload) => console.error("Gimme dat payload", payload);
const path =  '' ///TBD, see below
const client = amb.getClient();
const channel = client.getChannel(path);
channel.subscribe(callback);
```

To turn off the subscriber, hang on to the channel reference and `unsubscribe`
```js
channel.unsubscribe();
```

Finally, we need only to sort out how to communicate with the server what transactions we want to observe, exactly. Herein lies the secret sauce of `amb`. Here's [the util](https://github.com/nuvolo/sn-socket/blob/master/src/utils.ts) that we can look at to see how to populate the path

```js
function getEncodedFilter(filter: string): string {
  return btoa(filter).replace(/=/g, '-');
}

function getTablePath(table: string, filter: string): string {
  const encoded = getEncodedFilter(filter);
  return `/rw/default/${table}/${encoded}`;
}

function getSocketPath(params: SNSocketParams): string {
  const { table, filter } = params;
  return getTablePath(table, filter);
}

export { getSocketPath };
```

The `filter` is none than the `sysparm_query` string that you can get from your SN list view (right-click on the Filter Builder of any SN List, copy query, profit). This query string is base64-encoded, with `=` replaced with `-`. This provides the unique `"path"` which `amb` translates into a subscription. 

Add the `table` and the relative path to the socket handler, and the socket is registered. For example, you might do something like this

```js
const table = 'incident';
const query = 'active=true';
const encoded = "YWN0aXZlPXRydWU-" // result of getEncodedFilter(query);
const path = `/rw/default/${table}/${encoded}`

const callback = (payload) => console.error("Gimme dat payload", payload);
const client = amb.getClient();
const channel = client.getChannel(path);
channel.subscribe(callback);
```

Now make update to the `incident` table and see the magic!

![darkside](https://media.giphy.com/media/8SxGru3XzElqg/giphy.gif)


## Conclusion

So there you have it. Native socket functionality without any sniff of Angular.js (looking at you `recordWatcher`). As mentioned above, we have published a thin wrapper over `amb, and would welcome your feedback and contributions. You can see the [source code on Github](https://github.com/nuvolo/sn-socket). `sn-socket` wraps all of the things and exposes a `subscribe` method, for all of your Websocket needs. Let us know what you think.

In our last post of the series, we will see how Agent Workspace does a riff on this infrastructure and how you can leverage this information to bend the ServiceNow platform to your will. Until next time!
