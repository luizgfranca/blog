+++
title = 'Why not all your successful fetch calls might have actually hit your server'
date = 2025-06-19T16:59:50-03:00
draft = false
+++

When you see this code snippet, how many network calls do you think will be made when it runs in the browser? (assume any request made will be successful).

```js
try {
	const response1 = await fetch('http://localhost:3000/events');
	const data1 = await response1.json();
	console.log('First request response:', data1);
	
	const response2 = await fetch('http://localhost:3000/events');
	const data2 = await response2.json();
	console.log('Second request response:', data2);
	
} catch (error) {
	console.error('Error making requests:', error);
}
```

Surprisingly, it could be two, one, or even none!

That's because the browser will decide whether or not to send the requests depending on previous responses it received from your backend.

## How does this happen?

The backend (or any server that might be running it) can add the `Cache-Control` header with some value that tells the browser to use internal cache, for example. `Cache-Control: max-age=10` makes any subsequent calls in the next 10 seconds use cache instead of hitting the backend again.

This is expected behavior for most resources, but what might surprise you is that this doesn't only work for HTML pages, images, or JS files, but for `fetch` calls too.

For example, if we implement a backend controller to respond to `/event/latest` generating a unique UUID for each call:

```js
app.get('/event/latest', (req, res) => {
    const key = uuid.v4();
    res.json({ key })
})
```

And we run the code above in the browser, we'll get the expected behavior.

![image](img20250618210010.png) 
(notice we got 2 different IDs).

But if we make a small adjustment to the backend:

```js
app.get('/event/latest', (req, res) => {
    const key = uuid.v4();
    res.header('Cache-Control', 'max-age=60')
    res.json({ key })
})
```

We'll get this result:
![image](img20250618210451.png)

And if we reload the page before the next 60 seconds: 
![image](img20250618210519.png)

Looking at the network tab we'll see 4 requests: 
![image](img20250618210735.png)
but looking at the last 3... 

![image](img20250618210836.png) 
(note the "from disk cache").

This isn't behavior exclusive to this header combination - any cache header rules (which can be found at https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control) will work with `fetch`. More caching concepts can be found at https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Caching.

## Why does this happen?

With the modernization of web technologies in the 2010s, one of the W3C's initiatives was formalizing network resource fetching behavior, which wasn't always handled the same way across browsers. This culminated on the generalized concept of `fetch` for `resources`, described in the `fetch` spec (https://fetch.spec.whatwg.org/).

One of its results was the creation of a function for web applications that exposes this low-level concept. And this was the origin of the `fetch()` API that we can call from JavaScript, which was introduced in 2015 with ECMAScript 6 (ES6).

Because of this origin, it represents the same concept of fetching resources from a web service that's used by other browser features that do the same thing, like fetching HTML documents, images, JS, etc. That's why it shares the same behaviors, including HTTP caching, which makes this behavior happen.

---

To reproduce the experiments shown here, the code used is available in this GitHub repository: https://github.com/luizgfranca/fetch-cache-example
