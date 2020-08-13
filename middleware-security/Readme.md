# Middleware and Security

In this section we will be talking about middleware and security. The topic may seem a little unrelated but implementation wise they go hand in had.

## Middleware

Middleware is a function that wraps our handler. Thats all. 

```go
func Middleware(h handler) handler
```

This simple implementation has alot of power. In go functions can be passed in to other functions as a parameter. 

Say we want to add a log to every request we are serving that prints out the URL of the request. 

We can do that by creating our own middleware like so.

```go
func Logger(next http.Handler) http.Handler {
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    log.Println(r.URL)
    next.ServeHTTP(w, r)
  })
}
```

and then wrapping our main router

```go
log.Fatal(http.ListenAndServe(":"+port, Logger(r)))
```

On any request made to our server it will now print out the url of request on every request.

## Chaining Middlewares

Middlewares are useful tools and often one is not enough. If we want to add more of these middlewares we can keep on wrapping our router with our middlewares.

For example

```go
...SomeOtherMiddlerware(OtherMiddlerWare(Logger(r)))
```

There is couple of other ways to do this.

```go
// Chain applies middlewares to a http.HandlerFunc
func Chain(f http.Handler, middlewares ...func(next http.Handler) http.Handler) http.Handler {
  for _, m := range middlewares {
    f = m(f)
  }
  return f
}
```

Then we can use our middlewares like this,

```go
Chain(r, Logger, OtherMiddlerWare, SomeOtherMiddlerware, ...)
```

This in my opinion is a bit cleaner implementation.

In gorilla mux we have another option to do this.

Router has a method  `Use` that takes an array of Middlewares. This is useful to add middlewares to subrouters.

## Mux Handlers

Gorilla mux has a module named handlers. Its a collection of useful middleware handlers for gorilla mux. 

We can make use of one of these middlewares to make a better logger for our routes.

```go
func MuxLogger(next http.Handler) http.Handler {
  return handlers.LoggingHandler(os.Stdout, next)
}
```

We add this middleware in our chain

```go
  log.Fatal(http.ListenAndServe(":"+port, Chain(r, MuxLogger, Logger)))
```

And the output we get is,

```bash
2020/08/07 03:44:59 /api/v1/users
2020/08/07 03:44:59 /api/v1/users 901.821µs
::1 - - [07/Aug/2020:03:44:59 -0400] "GET /api/v1/users HTTP/1.1" 200 96107
```

The first line comes from our `Logger` middleware. 
The second line is from the `Time` middleware.
And finally the third line is from the `MuxLogger` that is using the `LoggingHandler` middleware from mux.

>The mux Logger middleware gets the http status code of the response being sent out. If we wanted, we could have implemented something like that ourselves. We would need to use a http hijacker and a custop response writer.