---
title: "Overcoming browser same origin policy"
category: taskcluster
comments: true
tags: [taskcluster]
---

One of my goals for 2016 Q1 was to write a
[monitoring dashboard](https://tools.taskcluster.net/status/) for Taskcluster.
It basically pings Taskcluster services to check if they are alive and also
acts as a feed aggregator for services Taskcluster depends on. One problem with
this approach is the
[same origin policy](https://en.wikipedia.org/wiki/Same-origin_policy), in which
web pages are only allowed to make requests to their own domain. For web servers
which is safe to make these cross domain requests, they can either implement
[jsonp](https://en.wikipedia.org/wiki/JSONP) or
[CORS](https://en.wikipedia.org/wiki/Cross-origin_resource_sharing). CORS is the
preferred way so we will focus on it for this post.

## Cross-origin resource sharing

CORS is a mechanism that allows the web server tell the browser that is safe to
accomplish a cross domain request. It consists of a set of HTTP headers with details
for the conditions to accomplish the request. The main response header is
`Access-Control-Allow-Origin`, which contains either a list of allowed domains or
a `*`, indicating any domain can make a cross request to this server. In a CORS
request, only a small set of headers is exposed to the response object. The server
can tell the browser to expose additional headers through the
`Access-Control-Expose-Headers` response header.

But what if the web server doesn't implement CORS? The only solution is to provide
a proxy that will make the actual request and add the CORS headers.

## cors-proxy

To allow the monitoring dashboard make requests for status state on remote services
that do not implement CORS, we created the
[cors-proxy](https://github.com/taskcluster/cors-proxy). It exports a `/request`
endpoint that allows you to make requests to any remote host. cors-proxy redirects
it to the remote URL and sends the responses back, with appropriate CORS headers set.

Let's see an example:

```javascript
$.ajax({
  url: 'http://cors-proxy.taskcluster.net/request',
  method: 'POST',
  contentType: 'application/json',
  data: {
    url: 'https://queue.taskcluster.net/v1/ping',
  }
}).done(function(res) {
  console.log(res);
});
```

The information about the remote request is sent in the proxy request body. All
parameter fields are shown in the project page.

Before you think on using the hosted server to bypass your own requests, cors-proxy
only honors requests from a
[whitelist](https://github.com/taskcluster/cors-proxy/blob/master/server.js#L12-L15).
So, only some subdomains under Taskcluster domain can use cors-proxy.

