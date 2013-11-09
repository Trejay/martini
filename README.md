# Martini [![Build Status](https://drone.io/github.com/codegangsta/martini/status.png)](https://drone.io/github.com/codegangsta/martini/latest)

Martini is a powerful package for quickly writing modular web applications/services in Golang.

~~~ go
package main

import "github.com/codegangsta/martini"

func main() {
  m := martini.Classic()
  m.Get("/", func() string {
    return "Hello world!"
  })
  m.Run()
}
~~~

Install the package:
~~~
go get github.com/codegangsta/martini
~~~

## Table of Contents
* [Martini](#martini-)
  * [Table of Contents](#table-of-contents)
  * [Classic Martini](#classic-martini)
    * [Handlers](#handlers)
    * [Routing](#routing)
    * [Services](#services)
    * [Serving Static Files](#serving-static-files)
  * [Middleware Handlers](#middleware-handlers)
    * [Next()](#next)
    * [Injecting Services](#injecting-services)

## Classic Martini
To get up and running quickly, `martini.Classic()` provides some reasonable defaults that work well for most web applications:
~~~ go
  m := martini.Classic()
  // ... middleware and routing goes here
  m.Run()
~~~

Below is some of the functionality `martini.Classic()` pulls in automatically:
  * Request/Response Logging - [martini.Logger](http://godoc.org/github.com/codegangsta/martini#Logger)
  * Panic Recovery - [martini.Recovery](http://godoc.org/github.com/codegangsta/martini#Recovery)
  * Static File serving - [martini.Static](http://godoc.org/github.com/codegangsta/martini#Static)
  * Routing - [martini.Router](http://godoc.org/github.com/codegangsta/martini#Router)

### Handlers
Handlers are the heart and soul of Martini. A handler is basically any kind of callable function:
~~~ go
m.Get("/", func() {
  println("hello world")
}
~~~

#### Return Values
If a handler returns a `string`, Martini will write the result to the current `*http.Request`:
~~~ go
m.Get("/", func() string {
  return "hello world" // HTTP 200 : "hello world"
})
~~~

#### Service Injection
Handlers are invoked via reflection. Martini makes use of *Dependency Injection* to resolve dependencies in a Handlers argument list. **This makes Martini completely  compatible with golang's `http.HandlerFunc` interface.** 

If you add an argument to your Handler, Martini will search it's list of services and attempt to resolve the dependency via type assertion:
~~~ go
m.Get("/", func(res http.ResponseWriter, req *http.Request) { // res and req are injected by Martini
  res.WriteHead(200) // HTTP 200
})
~~~

The following services are included with `martini.Classic()`:
  * [*log.Logger](http://godoc.org/log#Logger) - Global logger for Martini.
  * [martini.Context](http://godoc.org/github.com/codegangsta/martini#Context) - http request context.
  * [martini.Params](http://godoc.org/github.com/codegangsta/martini#Params) - `map[string]string` of named params found by route matching.
  * [http.ResponseWriter](http://godoc.org/net/http/#ResponseWriter) - http Response writer interface.
  * [*http.Request](http://godoc.org/net/http/#Request) - http Request.

### Routing
In Martini, a route is an HTTP method paired with a URL-matching pattern.
Each route can take one or more handler methods:
~~~ go
m.Get("/", func() {
  // show something
}

m.Post("/", func() {
  // create something
}

m.Put("/", func() {
  // replace something
}

m.Delete("/", func() {
  // destroy something
}
~~~

Routes are matched in the order they are defined. The first route that
matches the request is invoked.

Route patterns may include named parameters, accessible via the `martini.Params` service:
~~~ go
m.Get("/hello/:name", func(params martini.Params) string {
  return "Hello " + params["name"]
})
~~~

Route handlers can be stacked on top of each other, which is useful for things like authentication and authorization:
~~~ go
m.Get("/secret", authorize, func() {
  // this will execute as long as authorize doesn't write a response
})
~~~

### Services
Services are objects that are available to be injected into a Handler's argument list. You can map a service on a *Global* or *Request* level.

#### Global Mapping
A Martini instance implements the inject.Injector interface, so mapping a service is easy:
~~~ go
db := &MyDatabase{}
m := martini.Classic()
m.Map(db) // the service will be available to all handlers as *MyDatabase
// ...
m.Run()
~~~

#### Request-Level Mapping
Mapping on the request level can be done in a handler via `martini.Context`:
~~~ go
func MyCustomLoggerHandler(c martini.Context, req *http.Request) {
  logger := &MyCustomLogger{req}
  c.Map(logger) // mapped as *MyCustomLogger
}
~~~

#### Mapping values to Interfaces
One of the most powerful parts about services is the ability to map a service to an interface. For instance, if you wanted to override the `http.ResponseWriter` with an object that wrapped it and performed extra operations, you can write the following handler:
~~~ go
func WrapResponseWriter(res http.ResponseWriter, c martini.Context) {
  rw := NewSpecialResponseWriter(res)
  c.MapInterface(rw, (*http.ResponseWriter)(nil)) // override ResponseWriter with our wrapper ResponseWriter
}
~~~

### Serving Static Files
A `martini.Classic()` instance automatically serves static files from the "public" directory in the root of your server.
You can serve from more directories by adding more `martini.Static` handlers.
~~~ go
m.Use(martini.Static("assets")) // serve from the "assets" directory as well
~~~

## Middleware Handlers
### Next()
