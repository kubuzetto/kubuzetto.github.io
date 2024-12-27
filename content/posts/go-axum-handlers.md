+++
title = "Axum-style Magic Handler Functions in Go, Part 1"
date = "2024-12-14T15:30:22+03:00"
keywords = ['axum', 'go', 'http']
description = """Rustaceans using the axum framework can employ the "magic function" pattern to write
very descriptive handler functions with little boilerplate. Can we imitate it in Go? Let's find out!"""
showFullContent = false
readingTime = false
hideComments = true
toc = true
+++

> [Part 2 is out!](/posts/go-axum-handlers-pt2/)

## Introduction

Rustaceans using the axum framework can employ the
["magic function" pattern](https://github.com/alexpusch/rust-magic-patterns/tree/master/axum-style-magic-function-param)
to write very descriptive handler functions with little boilerplate.

Can we imitate it in Go? Let's find out!

### Why, though?

```
¯\_(ツ)_/¯
```

### Axum's magic functions

[axum](https://github.com/tokio-rs/axum) is a Rust web framework that focuses on ergonomics and modularity.
One particularly impressive feature of axum is that the signature of handler functions can declaratively
dictate how the request is parsed, and how the response is constructed:

```rust
// in main()
let app = Router::new().route("/users", post(create_user));

async fn create_user(Json(payload): Json<CreateUser>) -> (StatusCode, Json<User>) {
	(StatusCode::CREATED, Json(User { id: 1337, username: payload.username }))
}
```

The `Json<CreateUser>` argument is an _extractor_; telling axum that the request body is JSON
and should be deserialized as the `CreateUser` struct. Output is `Json<User>`, which serializes
the response body from the `User` struct to the JSON format. There is no imperative boilerplate
code for these operations.

Alex Puschinsky elegantly demystifies how it works
[here](https://github.com/alexpusch/rust-magic-patterns/tree/master/axum-style-magic-function-param),
TL;DR each extractor implements the `FromContext` trait to describe how to extract it from the request.

These functions are very flexible; you can extract some JSON from the body, other fields from the query,
and get access to the database instance from the state at the same time:

```rust
async fn get_products(State(db): State<Db>, Query(query): Query<CompanyInfo>, Json(body): Json<ProductFilters>) -> String {
	...
}
```

## The way Go does it

Compare the first example to the vanilla Go code that does the same thing:

```go
// in main()
mux := http.NewServeMux()
mux.HandleFunc("POST /users", createUser)

func createUser(w http.ResponseWriter, r *http.Request) {
	// parse input arguments
	var payload CreateUser
	defer func() { _ = r.Body.Close() }()
	if err := json.NewDecoder(r.Body).Decode(&payload); err != nil {
		w.WriteHeader(http.StatusBadRequest)
		return
	}

	// business logic
	code, user := http.StatusCreated, User{ID: 1337, Username: payload.Username}
	
	// write the response
	w.WriteHeader(code)
	_ = json.NewEncoder(w).Encode(user)
}
```

I mean, this is _not awful_.

I cut some corners in terms of observability and proper error handling for the
sake of brevity, but still most of the function is boilerplate.

### A minor improvement

Before I move on to other things, there is a small helper function we can
immediately extract here. We will use it later, and it's better to address that
before things get complicated. See the `defer` statement there? That will be delayed
until the end of the handler; instead of closing the body right after it's read.
Let's put the decoding logic in its own function then:

```go
func createUser(w http.ResponseWriter, r *http.Request) {
	var payload CreateUser
	if err := decodeBodyWith(r, json.NewDecoder, &payload); err != nil {
		w.WriteHeader(http.StatusBadRequest)
		return
	}
	code, user := http.StatusCreated, User{ID: 1337, Username: payload.Username}
	w.WriteHeader(code)
	_ = json.NewEncoder(w).Encode(user)
}

func decodeBodyWith[D interface {
	Decode(any) error // duck-typing *json.Decoder, *xml.Decoder etc.
}](r *http.Request, newDecoderFn func(io.Reader) D, dest any) (err error) {
	if r.Body != nil {
		defer func() {
			// also join any possible errors from closing the body
			err = errors.Join(err, r.Body.Close())
		}()
		err = newDecoderFn(r.Body).Decode(&dest)
	}
	return
}
```

Not a huge win for now, but we'll make use of this function later.

## Where we want to end up

Ideally, we want our `createUser` function to contain the business logic only:

```go
func createUser(payload JSON[CreateUser]) (response JSON[User], _ error) {
	response.V = User{ID: 1337, Username: payload.V.Username}
	return
}
```

> The `JSON` type is as follows:
> ```go
> type JSON[Inner any] struct{ V Inner }
> ```
> Note that we cannot _embed_ `Inner` here; Go is not (yet?) capable of this.
> We have to put `Inner` as a member field, so I chose a short name at least.
> We will implement `JSON`'s behavior a little later.

We need a higher-order function to wrap and transform our custom-signature handlers into `http.HandlerFunc`s.

## The tools at hand

The implementation in `axum` makes good use of Rust concepts like generics,
traits and associated types. In Go, we have interfaces instead of traits.
Our generics functionality is limited; methods cannot be generic on an extra
type, type erasure does not exist, constraints suck.

So, right off the bat, we want to write a wrapper function that will wrap our custom-signature
handlers, right? There isn't an in-built `Func` constraint; so we can't do something like:

```go
func Handler(handler Func) http.HandlerFunc { ... }
```

> We cannot describe it ourselves either. If you try to define a constraint like:
> ```go
> type Func[R, X, Y, Z, T any] interface {
> 	func(X) (R, error) |
> 	func(X, Y) (R, error) |
> 	func(X, Y, Z) (R, error) |
> 	func(X, Y, Z, T) (R, error) // let's say 4 is enough for now
> }
> ```
> then you _have_ to spell out _all_ of `[R, X, Y, Z, T]` _everywhere_ you try to use it.
> Go cannot infer them for you; because if you pass `func(int) (string, error)` here; what are `Y, Z, T`?
> You can't omit them either, because there is no type erasure in Go.

So we immediately steer away from generics here. It is to be used sparingly at best.

Another option would be code generation, but I'd rather not introduce tools to the build pipeline right now.
Dealing with Go AST also kind of sucks the fun out of this kind of self-challenge; it feels like accepting defeat.

Our third option is to switch to the dark side, and embrace reflection. I _hate_ reflection, but I can't escape from it.
Many essential Go functionalities (including JSON encoding/decoding) use it anyway, so let's give it a try.

## Calling a function with reflection

With reflection, we should get the `handler` type as `any`, so the first check should be,
"is this type _really_ a function"?

```go
func Handler(handler any) http.HandlerFunc {
	fnVal := reflect.ValueOf(handler)
	if fnVal.Kind() != reflect.Func {
		panic(PanicReasonHandlerExpectsAFunc)
	}
	...
}

const PanicReasonHandlerExpectsAFunc = "Handler parameter should be a function"
```

Now that we have a `reflect.Value` in our hands, we can invoke the function using:

```go
// reflect/value.go
func (v Value) Call(in []Value) []Value
```

So, the flow becomes:

1. Construct each input and wrap them into `Value`s,
2. `Call` the function using reflection,
3. Convert the output `Value` slice into expected types.
4. Render the response using the output data.

```go
func Handler(handler any) http.HandlerFunc {
	fnVal := reflect.ValueOf(handler)
	if fnVal.Kind() != reflect.Func {
	    panic(PanicReasonHandlerExpectsAFunc)
	}
	fnType := fnVal.Type()
	extractInputs := toExtractorFn(fnType)                 // 0
	convertOutputs := toOutputHandlerFn(fnType)            //
	return func(w http.ResponseWriter, r *http.Request) {
	    var response any
	    inputs, err := extractInputs(r)                    // 1
	    if err == nil {
	        outputs := fnVal.Call(inputs)                  // 2
	        response, err = convertOutputs(outputs)        // 3
	    }
	    if err == nil {
	        _ = WriteResponse(w, response)                 // 4
	    } else {                                           //
	        _ = writeErrResp(w, err)                       //
	    }
	}
}
```

The 0th step above is preparing the input/output conversion functions ahead-of-time.
We can do this because we have the handler function's type before it is actually called.
We can also make type verifications in this stage. It is a good place to start from.

Since we can get any number of inputs, let's first handle the output conversion.

## Converting the outputs

We expect the function's output to either be an `error`, or `(T, error)`; meaning
an arbitrary response type (that we will receive as `any`) and an `error`. Any other
output is unexpected, and should cause a panic.

```go
func toOutputHandlerFn(fnType reflect.Type) func([]reflect.Value) (any, error) {
	switch numOut := fnType.NumOut(); {
	case numOut == 1 && fnType.Out(0).Implements(errType):
		return handleOneOutput // func(*) error
	case numOut == 2 && fnType.Out(1).Implements(errType):
		return handleTwoOutputs // func(*) (T, error)
	default:
		panic(PanicReasonHandlerUnexpectedNumberOfReturns)
	}
}

var errType = reflect.TypeFor[error]()

const PanicReasonHandlerUnexpectedNumberOfReturns = "Handler should return either error, or (T, error)"

func handleOneOutput(v []reflect.Value) (_ any, err error) {
	err, _ = v[0].Interface().(error)
	return
}

func handleTwoOutputs(v []reflect.Value) (any, error) {
	err, _ := v[1].Interface().(error)
	return v[0].Interface(), err
}
```

Now we have a function that returns our handler's output to `(any, error)`.

## Parsing the inputs

Implementing the extractors for the input parameters is a whole other story.
First of all, we define the extractor behavior as an interface:

```go
type Extractor interface {
	Extract(*http.Request) (any, error)
}
```

This interface is _unusual_, and I'll tell you why in a moment. But let's first try to use it.

We should iterate each of the function type's inputs, and verify that they implement the `Extractor` interface.
The method might have a pointer receiver, so we'll handle that case as well.
Since it's so common, I also accept `context.Context` as a valid argument here. It will be
filled with the request's context.

```go
func toExtractorFn(fnType reflect.Type) func(*http.Request) ([]reflect.Value, error) {
	numIn := fnType.NumIn()
	funcs := make([]func(*http.Request) (reflect.Value, error), numIn)
	for i := range numIn {
		// extType is an interface; do the check with Implements
		if arg := fnType.In(i); arg.Implements(extType) {
			funcs[i] = extractFuncOfType(arg)
		// also check the pointer of the type for implementing Extractor
		} else if argPtr := reflect.PointerTo(arg); argPtr.Implements(extType) {
			funcs[i] = extractFuncOfType(argPtr)
		// accept context.Context as a valid argument type as well
		} else if arg.Implements(ctxType) {
			funcs[i] = extractCtx
		// anything else is grounds for a panic
		} else {
			panic(PanicReasonUnknownArgType)
		}
	}
	// return a function that extracts ALL inputs at once
	return func(r *http.Request) (values []reflect.Value, err error) {
		values = make([]reflect.Value, numIn)
		for i := 0; err == nil && i < numIn; i++ {
			values[i], err = funcs[i](r)
		}
		return
	}
}

func extractFuncOfType(arg reflect.Type) func(*http.Request) (reflect.Value, error) {
	zero := reflect.Zero(arg).Interface().(Extractor)
	return func(r *http.Request) (reflect.Value, error) {
		v, err := zero.Extract(r)
		return reflect.ValueOf(v), err
	}
}

func extractCtx(r *http.Request) (reflect.Value, error) {
	return reflect.ValueOf(r.Context()), nil
}

var extType = reflect.TypeFor[Extractor]()
var ctxType = reflect.TypeFor[context.Context]()

const PanicReasonUnknownArgType = "Cannot determine how to extract handler argument"
```

## Writing the response

> I'm sure there are better, less fragile ways of doing this. This part
> bores me because it's not strictly a part what we are trying to achieve,
> so I'll half-ass it and write the shortest version that I can.

We want individual types to be able to control how they are written into the response.
Also, our original example returned `201` instead of `200`; so we want to be able to change
the status code based on the type of success. Finally, our handlers should be able to take
wrapper types into account. Something like this will do for now:

```go
type Responder interface {
	Response(http.ResponseWriter) error
}

type StatusCoder interface{ StatusCode() int }

func WriteResponse(w http.ResponseWriter, resp any) (err error) {
	if s, ok := resp.(Responder); ok {
		return s.Response(w)
	}
	return WriteJSONResponse(w, resp, StatusCodeFrom(resp))
}

func StatusCodeFrom(resp any) (code int) {
	if s, ok := resp.(StatusCoder); ok {
		code = s.StatusCode()
	}
	return
}

func WriteJSONResponse(w http.ResponseWriter, resp any, code int) error {
	w.Header().Set("Content-Type", "application/json")
	if code > 0 {
		w.WriteHeader(code)
	}
	return json.NewEncoder(w).Encode(resp)
}
```

Nothing much to comment on, really. If the type is `Responder`, use the
custom response function. Otherwise, write it as JSON. Both functions are
public because we want to use these in other places soon.

Similarly, error cases should be able to dictate their own error codes:

```go
type errResp struct { Error string `json:"error"` }

type errWithCode struct {
	code int
	error
}

func writeErrResp(w http.ResponseWriter, err error) error {
	statusCode := http.StatusInternalServerError
	if coded := new(errWithCode); errors.As(err, coded) {
		statusCode = coded.code
	}
	return WriteJSONResponse(w, errResp{Error: err.Error()}, statusCode)
}

func WithStatusCode(err error, code int) error {
	if err != nil && code > 0 {
		err = errWithCode{code: code, error: err}
	}
	return err
}
```

For example, with this we can modify `toExtractorFn` to return `400`
when the arguments cannot be extracted:

```go
return func(r *http.Request) (values []reflect.Value, err error) {
	values = make([]reflect.Value, numIn)
	for i := 0; err == nil && i < numIn; i++ {
		values[i], err = funcs[i](r)
	}
	// this line is added
	err = WithStatusCode(err, http.StatusBadRequest)
	return
}
```

## Implementing the JSON type

Let's implement the `JSON`'s extractor functionality:

```go
func (v *JSON[T]) Extract(r *http.Request) (any, error) {
	err := decodeBodyWith(r, json.NewDecoder, &v.V)
	return v, err
}
```

We already have the `decodeBodyWith` helper, so the implementation is trivial.

### Testing things so far

Time for a smoke test!

```go
func TestHandler(t *testing.T) {
	mux := http.NewServeMux()
	mux.HandleFunc("POST /users", Handler(createUser))
	// success case
	req, _ := http.NewRequest(http.MethodPost, "/users",
		strings.NewReader(`{"username": "abc"}`))
	resp := httptest.NewRecorder()
	mux.ServeHTTP(resp, req)
	fmt.Printf("%d: %s\n", resp.Code, resp.Body.String())
	// error case
	req, _ = http.NewRequest(http.MethodPost, "/users",
		strings.NewReader(`{{`))
	resp = httptest.NewRecorder()
	mux.ServeHTTP(resp, req)
	fmt.Printf("%d: %s\n", resp.Code, resp.Body.String())
}
```

If we-

```
=== RUN   TestHandler
--- FAIL: TestHandler (0.00s)
panic: runtime error: invalid memory address or nil pointer dereference [recovered]
	panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x2 addr=0x0 pc=0x102f2e594]
```

...oh. Smoke.

### The problem with Extractor

Let's take another look at the `Extractor` interface.

```go
type Extractor interface {
	Extract(*http.Request) (any, error)
}
```

It defines a _method_ for a type that _constructs_ that type. How can we call the method of a value that doesn't yet
exist?

Well, this is how we _did_ call it:

```go
func extractFuncOfType(arg reflect.Type) func(*http.Request) (reflect.Value, error) {
	zero := reflect.Zero(arg).Interface().(Extractor)
	return func(r *http.Request) (reflect.Value, error) {
		v, err := zero.Extract(r)
		return reflect.ValueOf(v), err
	}
}
```

Were it not for the reflection boilerplate, this is basically equivalent to:

```go
var zero T
value, err := any(zero).(Extractor).Extract(r)
```

Or in our example;

```go
value, err := (*JSON[T])(nil).Extract(r)
```

__We get a panic because `v` is `nil`__.

> Normally, this is a kind of function that would be defined on the __type__, instead of
> an __instance__ of the type. Rust allows this through traits; but Go does not have the mechanism.

We could "fix" this by making the receiver function pass-by-value instead. However,
the extractor functions are reused, and so the zero value passed to them is never garbage collected.
That's why we want to use the pointer, because `T` can be an arbitrarily large struct.

Instead, we must be careful __not__ to perform read/write operations on the receiver instance.
Which is not that hard normally; just don't even name the receiver in the method:

```go
// keep the receiver unnamed, and we won't run the risk of using it
func (*JSON[T]) Extract(r *http.Request) (any, error) {
	var v JSON[T] // create a new JSON[T], and work on that
	return v, decodeBodyWith(r, json.NewDecoder, &v.V)
}
```

If we try again...

```
=== RUN   TestHandler
200: {"V":{"id":1337,"username":"abc"}}

400: {"error":"invalid character '{' looking for beginning of object key string"}

--- PASS: TestHandler (0.00s)
PASS
```

The test passes!

> Well _of course_ the test passes; we have `Printf`s instead of assertions. But you get what I mean.

### Rendering the response properly

...except the output is a bit weird. We want the `JSON` type to be transparent.

That's an easy fix though:

```go
func (v JSON[T]) Response(w http.ResponseWriter) error {
	return WriteJSONResponse(w, v.V, StatusCodeFrom(v.V))
}
```

Also, we want the success case to return `201` instead:

```go
func (User) StatusCode() int { return http.StatusCreated }
```

Finally we have:

```
=== RUN   TestHandler
201: {"id":1337,"username":"abc"}

400: {"error":"invalid character '{' looking for beginning of object key string"}

--- PASS: TestHandler (0.00s)
PASS
```

## Summary

Man, we wrote a lot of code! So perhaps it's better to list only the user code
here, hiding the functionality that would normally be a part of a library:

```go
func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("POST /users", Handler(createUser))
	...
}

func createUser(payload JSON[CreateUser]) (response JSON[User], _ error) {
	response.V = User{ID: 1337, Username: payload.V.Username}
	return
}

type CreateUser struct {
	Username string `json:"username"`
}

type User struct {
	ID       uint   `json:"id"`
	Username string `json:"username"`
}

func (User) StatusCode() int { return http.StatusCreated }

type JSON[Inner any] struct{ V Inner }

func (*JSON[T]) Extract(r *http.Request) (any, error) {
	var v JSON[T]
	return v, decodeBodyWith(r, json.NewDecoder, &v.V)
}

func (v JSON[T]) Response(w http.ResponseWriter) error {
	return WriteJSONResponse(w, v.V, StatusCodeFrom(v.V))
}
```

Now we can implement other extractors like `Headers`, `Query` and so on:

```go
type Headers[Inner any] struct { V Inner }

// usage
var _ = Headers[struct {
    X string `header:"x"`
    Y int    `header:"y"`
}]
```

I won't show the implementations for these, because the post is getting long. Long story short, we want to
check that `Inner` is a struct, then iterate its fields for the `header` tag and extract the relevant header
from the request. The same logic appears in the
[gin library](https://github.com/gin-gonic/gin/blob/master/binding/header.go#L19).

Similarly, I think we can implement a `State` type that fetches custom
data from the request's context, to implement things like `DB`.

### A final touch

In Rust, you can make use of RAII and implement the `Drop` trait for your extractors when they need cleaning up.
In Go, we can use a `Close() error` function, provided by the `io.Closer` interface. We simply have to check for all
input types at the end of the handler like this:

```go
if err == nil {
    _ = WriteResponse(w, response) // 4
} else { //
    _ = writeErrResp(w, err) //
}
// code below is added
for _, e := range inputs {
    if e.IsValid() { // must check if the value is initialized
        if c, ok := e.Interface().(io.Closer); ok {
            _ = c.Close()
        }
    }
}
```

Here is an example extractor that has a closer:

```go
type Logger struct{ *slog.Logger }

func (*Logger) Extract(r *http.Request) (any, error) {
	log := slog.Default().With("route", r.URL.Path, "request_id", uuid.NewString())
	log.Info("endpoint start")
	return Logger{Logger: log}, nil
}

func (l Logger) Close() error {
	l.Logger.Info("endpoint end")
	return nil
}

...

func createUser(l Logger, payload JSON[CreateUser]) (response JSON[User], _ error) {
	response.V = User{ID: 1337, Username: payload.V.Username}
	l.Info("created", "user", payload.V.Username)
	return
}
```

We can see the logs:

```
=== RUN   TestHandler
2024/12/17 02:24:35 INFO endpoint start route=/users request_id=ce9f5c48-c9af-4492-a862-cb4ca94aadb5
2024/12/17 02:24:35 INFO created route=/users request_id=ce9f5c48-c9af-4492-a862-cb4ca94aadb5 user=abc
2024/12/17 02:24:35 INFO endpoint end route=/users request_id=ce9f5c48-c9af-4492-a862-cb4ca94aadb5
201: {"id":1337,"username":"abc"}

2024/12/17 02:24:35 INFO endpoint start route=/users request_id=16046b80-1de9-4f32-a242-03237aff81c6
2024/12/17 02:24:35 INFO endpoint end route=/users request_id=16046b80-1de9-4f32-a242-03237aff81c6
400: {"error":"invalid character '{' looking for beginning of object key string"}
```

That's all I have for this post. Go now.


---
[Discuss in r/programming](https://www.reddit.com/r/programming/comments/1hmy79l/axumstyle_magic_handler_functions_in_go/)
| [Discuss in r/golang](https://www.reddit.com/r/golang/comments/1hnrvcf/axumstyle_magic_handler_functions_in_go_part_1/)
