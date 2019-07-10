---
title: "Middlewares in Go"
date: 2019-06-17T17:29:20-02:00
draft: false
comments: true
---

When we create web application with Go, the first code smell that we found is code duplicity.

We always need to do some tasks on each endpoint like logging things, user authentication, send
data to NewRelic, etc. The thing that I fell more confortable to do things like that  without
duplicity is create Middlewares.

To show you an example of implementation, let's create the following structure:

```
/handlers
    - home.go
/middlewares
    - authentication.go
    - notFound.go
main.go
```

Let's start creating our home handler:

```go
package handlers

import (
    "net/http"
)

type HomeHandler struct {
    handler http.Handler
}

func (handler HomeHandler) ServeHTTP(response http.ResponseWriter, request *http.Request) {
    response.Write([]byte("Middlewares in Go!"))
}
```

As we can see, we defined the strucutre to receive an object that implements the `Handler` interface and we implemented the function `ServeHTTP` with our handler code, that will return a simple text message on the response. All of our handlers must follow these base structure only modifying the body of `SetveHTTP`
function.

Now we will create our two example middlewares:

The first will treat the not founded pages and return a 404 HTTP status code. 

```go
package middlewares

import (
	"net/http"
)

type NotFoundMiddleware struct {
	Next http.Handler
}

func (controller NotFoundMiddleware) ServeHTTP(response http.ResponseWriter, request *http.Request) {
	path := request.URL.Path
    routes := []string{
        "/",
        "/post",
    }

	for _, route := range routes {
		if route == path {
			controller.Next.ServeHTTP(response, request)
			return
		}
	}

	http.NotFound(response, request)
}
```

And the second will validate a JWT token and do the authorization of the request.

```go
package middlewares

import (
    "encoding/json"
    "fmt"
	"net/http"
    "strconv"
)

type AuthenticationMiddleware struct {
	Next http.Handler
}

func (controller AuthenticationMiddleware) ServeHTTP(response http.ResponseWriter, request *http.Request) {
	authHeader := request.Header.Get("Authorization")
	tokenString := strings.Replace(authHeader, "Bearer ", "", 1)

	token, err := jwt.Parse(tokenString, func(token *jwt.Token) (interface{}, error) {
		return []byte("MySecretTokenHere"), nil
	})

	if err == nil && token.Valid {
        	controller.Next.ServeHTTP(response, request)
		return
	} 
    
    	response.WriteHeader(http.StatusUnauthorized)
    	fmt.Fprintln(response, "Unauthorized")
}
```

Now we will create our main file, where we will put our middlewares in action and start the webserver.

```go
package main

import (
    "github.com/henriqueholanda/middlewares-go/handlers",
    "github.com/henriqueholanda/middlewares-go/middlewares",
    "net/http"
)

function main() {
    http.Handle(
        "/",
        middlewares.NotFoundMiddleware{handlers.HomeHandler},
    )
    http.Handle(
        "/post",
        middlewares.NotFoundMiddleware{middlewares.AuthenticationMiddleware{handlers.PostController}},
    )
    
    http.ListAndServe(":8085", nil)
}
```

This is our main file, it's the first to be executed, and we have all routes on them.
The first parameter of the route is the endpoint, and the second need to be an implementation of `http.Handler`, and in our case, we put our middlewares.

Each middleware receive the next one to be executed until arrive on the handler. 

The best thing on this approach is the possibility to define wich middleware will be executed on each endpoint of our application, in our example the "/" endponit don't need to do authentication based on JWT.

Now we need to start the webserver. To do this, just run the following code:
```bash
$ go run main.go
```

When you do a request to the webserver, it will enter on each `http.Handler` following the position that
it be on the code.

When you send a request to a route that not exists like http://localhost:8085/test, it will enter on the first router handler `/` and in the first middleware `NotFoundMiddleware`. Inside this middleware it will 
try to match the request route with the defined routes of our application. If it matches with some route,
it will execute the next handler, that can be another middleware or the application handler like `HomeHandler`. And if not matches, it will return a 404 status code on the response and it stop the
execution.

So, it was a simple example that how implements middlewares in Go. If you have some question, suggestion
or any kind of feedback, please message me, all help is welcome. 😉
