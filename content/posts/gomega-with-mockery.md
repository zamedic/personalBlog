---
title: "Gomega With Mockery"
date: 2023-04-20T14:31:47+02:00
tags:
  - golang
  - GoMega
  - Mockery
categories:
  - Software Development
---

# Using GoMega matchers with Mockery
I am a firm believer of using the right tool for the job. Within golang, we are lucky to have two awesome tools at
our disposal. 
* [Gomega](https://onsi.github.io/gomega/) - A matcher library for golang
* [Mockery](https://github.com/vektra/mockery) - A mock code generator for golang

Gomega has a lot of matchers that can be used to test your code. Mockery is a great tool to generate mocks for your interfaces.
I have been using both of these tools for a while now and I have found that they work really well together. In this post, I will
show you how to use Gomega matchers with Mockery.

So, lets work through an example. 

We have an interface called `RequestGenerator` that looks like this:

```go
type HeaderValidator interface {
	ValidateHeader(parameters http.Header) error
}
```
If the logic within ValidateHeader is simple enough, you might just call the real concrete implementation of the interface.
However, if the logic is more complex, you might want to mock the interface and test the logic in isolation, especially if you
want to control success vs error scenarios.

```bash
mockery -name=RequestGenerator --with-expecter
```

So, now we have a mock function we can call 

```go
validator.EXPECT().ValidateHeader(????????).Return(&r, nil)
```

Sure, we can just pass in a `http.Header` object, like this:

```go
validator.EXPECT().ValidateHeader(http.Header{
	"Content-Type": []string{"application/json"},
	"Accept":       []string{"application/json"},
}).Return(nil)
```

But, if the header parameters are in a different order, this will fail.
If the header parameter has additional fields, this will fail. 

We saw this when we created unit tests which would spin up a GinFizz http server, using mocked out dependencies, and use 
a real http client to call the endpoints, thereby testing the mappings and data bindings were correctly defined. 

When we did this, we found that the order of the headers was not consistent. This meant that our tests would fail randomly.
There are also additional header values with are not relevant to the test, but are added by the http client.
```
key = {string} "User-Agent"
0 = {string} "Go-http-client/1.1"
key = {string} "Content-Length"
0 = {string} "2"
key = {string} "Authorization"
0 = {string} "someAuthorization"
key = {string} "Content-Type"
0 = {string} "application/json"
key = {string} "X-Act-Tenantid"
0 = {string} "someTenant"
key = {string} "Accept-Encoding"
0 = {string} "gzip"
```

We had to find a way to test the header values, to ensure the values we were interested in were present, but ignore the rest.
Particularly, we wanted to ensure the Authorization and X-Act-Tenantid headers were present, but ignore the rest.

We could have used a custom matcher, but Gomega already has a matcher that does exactly what we want. 

```go
import (
	. "github.com/onsi/gomega"
   
)

...

And(
		HaveKeyWithValue("Authorization", []string{"someAuthorization"}),
		HaveKeyWithValue("X-Lcp-Act-Tenantid", []string{"someTenant"}),
	),
```
The above code will check that the map has the Authorization and X-Act-Tenantid keys, and that the values are correct.
However, this is a GoMegaMatcher and Mockery only has a argumentMatcher interface, which is private. This argumentMatcher
is exposed by one function,

```go
func MatchedBy(fn interface{}) argumentMatcher {
```

so, we can create a custom matcher that wraps the Gomega matcher.

```go
// MegaMatch is a helper function to use GoMega in conjunction with mockery. It takes a GoMega matcher and returns a
// function that can be used in the EXPECT() call of a mockery object.
func MegaMatch(matcher types.GomegaMatcher) interface{} {
	return mock.MatchedBy(func(x interface{}) bool {
		success, err := matcher.Match(x)
		if err != nil {
			panic(err)
		}
		if success {
			return true
		}
		panic(matcher.FailureMessage(x))
	})
}
```

and now use this function in our mock call

```go  
validator.EXPECT().ValidateHeader(
    MegaMatch(
        And(
            HaveKeyWithValue("Authorization", []string{"someAuthorization"}),
            HaveKeyWithValue("X-Lcp-Act-Tenantid", []string{"someTenant"}), 
		),
	)
).Return(&r, nil)

```







