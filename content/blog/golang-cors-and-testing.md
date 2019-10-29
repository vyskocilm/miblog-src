---
title: "Golang CORS and Testing"
date: 2018-12-18T19:27:15+02:00
image: "img/gopher-flicker.jpg"
draft: true
---

Working on a web project spread in more domains brings you to [Cross-Origin Resource Sharing](https://www.html5rocks.com/en/tutorials/cors/). Browsers use it to if it code from one origin can call HTTP methods placed elsewhere. Responses must contain specific headers, like [Access-Control-Allow-Origin](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Access-Control-Allow-Origin), which must match requesting [Origin](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Origin) of the site issuing the call. More information about CORS can be found at [html5rocks](https://www.html5rocks.com/en/tutorials/cors/).

CORS look like simple thing to develop. Just add a few HTTP headers for regular methods. Then you must handle `OPTIONS`, then you must know all the headers you're sending, figure out how will your application behave if `Origin` matches and if not. Therefor it is a good idea to use existing library doing so. For golang applications there is [github.com/rs/cors](https://github.com/rs/cors). `net/http` compatible middleware, which wraps all handlers and apply CORS rules based on configuration. Usage is very simple, one just create instance of `cors.Cors`, `http.ServeMux` and register handlers. The question is then - _how to test the CORS as a part of `testing` routine?_.

## Handler and HandlerFunc

HTTP handling is done in simple function with two mandatory arguments.

```
func(w http.ResponseWriter, r *http.Request)
```

where `http.ResponseWritter` is code providing methods writing HTTP reply and `http.Request` provides data from HTTP request. Functions are represented as _interface_ `http.Handler`. It has only one method

```
type Handler interface {
        ServeHTTP(ResponseWriter, *Request)
}
```

Beauty of this system is simplicity and the fact `ResponseWritter` is _interface_ enables unit testing via `httptest.NewRecorder`.

```
req := httptest.NewRequest("GET", "http://example.com/foo", nil)
w := httptest.NewRecorder()
handler(w, req)
```

## Handlers all the way down

The main trick is here

```
func Middleware(h http.Handler) http.Handler
```

`http.ServeMux` is new type we have not talked before. It registers more handling functions and dispatch them based on request URL. And because it implements `http.Handler` interface, it is easy to join it with `Middleware` function and make them working together. The integration is simply done by calling `ServeHTTP` method. This is _exactly_ how `cors.Cors` is integrated to your application, by the way.

## CORS and testing

As we have discussed before, golang supports HTTP handlers testing via `httptest.NewRecorder`. This implements `http.ResponseWritter` interface, so it is easy to run handler function. The main trick is to get CORS enabled `http.Handler` in the test and call it.

```
// main.go
cors = cors.Default()           // initialize global variable cors.Cors
mux = http.NewServeMux()        // initialize new serve mux

// main_test.go
r := http.NewRequest(....)
w := httptest.NewRecorder()
handler := cors.Handler(mux)    // THE TRICK!
handler.ServeHTTP(w, r)
```

## An example

Here is the full working example integrating CORS to simple http server. `ServeMux` and `cors.Cors` are now wrapped in `Srv` structure, so we can access the right handler in the test.

First, the proof, that CORS works well

```
$ curl -v -H Origin:http://example.com http://localhost:8000
*   Trying ::1...
* TCP_NODELAY set
* Connected to localhost (::1) port 8000 (#0)
> GET / HTTP/1.1
> Host: localhost:8000
> User-Agent: curl/7.62.0
> Accept: */*
> Origin:http://example.com
> 
< HTTP/1.1 200 OK
**< Access-Control-Allow-Origin: *
< Vary: Origin**
< Date: Tue, 18 Dec 2018 14:16:56 GMT
< Content-Length: 25
< Content-Type: text/html; charset=utf-8
< 
<html>Hello world</html>
* Connection #0 to host localhost left intact
```

And the source code + test.

<script src="https://gist.github.com/vyskocilm/3007ce0754d86b7e7b0754383b46b2a1.js"></script>

Major takeways

*   It is usually better to use battle tested code even for simple looking problem
*   `net/http` is simple yet powerful library providing great abstractions to build on top of
*   Testing is important and there should be always the way to write testable code
*   Working example of unit test for CORS integartion in golang application

Logo by samthor@Flickr: [https://www.flickr.com/photos/samthor/5994939587]
