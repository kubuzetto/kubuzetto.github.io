+++
title = "Axum-style Magic Handler Functions in Go, Part 2"
date = "2024-12-27T23:21:56+03:00"
keywords = ['axum', 'go', 'http']
description = """We continue with our aimless pursuit of a boilerplate-free Go. Now with more structs!"""
showFullContent = false
readingTime = false
hideComments = true
toc = true
+++

## Introduction

In the [first part](/posts/go-axum-handlers/) of this blog post, we tried to replicate the
[magic of axum](https://github.com/alexpusch/rust-magic-patterns/tree/master/axum-style-magic-function-param)
in Go. In the end, we had endpoint handlers that looked like this:

```go
func createUser(l Logger, payload JSON[CreateUser]) (response JSON[User], _ error) {
	response.V = User{ID: 1337, Username: payload.V.Username}
	l.Info("created", "user", payload.V.Username)
	return
}
```

I recommend you read that one first.

Today, we continue with our aimless pursuit of a boilerplate-free Go. Now with more structs!

Here's the thing: In our previous implementation, we had to invoke the function using reflection, because our handlers
can have any number of arguments. But reflection has some overhead; especially function calls this way tend to be
[pretty slow](https://github.com/golang/go/issues/7818).

### A way out

I'm thinking, maybe we can represent all function arguments as __fields of one struct__. Then our handler function
signature will be:

```go 
func handler[Args, Output any](a Args)(Output, error)
```

The previous example will end up looking like this:

```go
func createUser(p struct {
	Logger
	JSON[CreateUser]
}) (User, error) {
	p.Info("created", "user", p.V.Username)
	return User{ID: 1337, Username: p.V.Username}, nil
}
```

This signature has the big advantage that our `Handler` wrapper function can now represent the function as a
function type, instead of falling back to `any`. Sure, the input type must be filled using reflection again;
but the function invocation itself will be regular Go code. So let's start!

## Going through the motions

Once again, we create a function that extracts the inputs. Note that this time we don't have to reflectively work
on the outputs; because we settled on the function signature - one input struct, two outputs. `Output` is treated
as `any` in the function body, but is generic in the signature; that is for calling site readability and convenience.

1. Prepare a function to extract function inputs from the request; and into the input struct.
2. During handler calls; invoke this function. This time we return the constructed struct.
3. Handling the output of the function is the same as the previous version, I won't get into details here.

```go
func Handler[Args, Output any](handler func(Args) (Output, error)) http.HandlerFunc {
	extractInputs := toExtractorFn[Args]() // 1
	return func(w http.ResponseWriter, r *http.Request) {
		var out any
		args, err := extractInputs(r) // 2
		if err == nil {
			out, err = handler(args)
		}
		// 3
		if err == nil {
			_ = WriteResponse(w, out)
		} else {
			_ = writeErrResp(w, err)
		}
	}
}
```

All type checks and reflection shenanigans are on the input argument this time; so we don't have reflection calls here.
Let's get into `toExtractorFn`. First things first, we must check that `Args` is a struct:

```go
const PanicReasonHandlerExpectsAStruct = "Handler argument should be a struct"

func toExtractorFn[Args any]() func(*http.Request) (Args, error) {
	argType := reflect.TypeFor[Args]()
	if argType.Kind() != reflect.Struct {
		panic(PanicReasonHandlerExpectsAStruct)
	}
	...
```

Familiar stuff. This time, we will iterate the fields of `Args`; each field should implement an
`Extractor` interface like before.

```go
    ...
    numFields := argType.NumField()
    funcs := make([]func(*http.Request, *Args) error, numFields)
    for i := range numFields {
        field := argType.Field(i)
        if !reflect.PointerTo(field.Type).Implements(extType) {
            panic(PanicReasonUnknownFieldType)
        }
        funcs[i] = extractFieldOfType[Args](field.Type, field.Offset)
    }
    ...
}

var extType = reflect.TypeFor[Extractor]()

const PanicReasonUnknownFieldType = "Cannot determine how to extract handler argument field"
```

This will give us an array of functions that extract the arguments from the request.
Note how I use `reflect.PointerTo` here; that's because I want the types implementing the `Extractor`
interface to have _pointer receivers_. Our previous implementation also supported value receivers,
this time I don't bother with that case. The reason will be clear very soon.

What arguments should these functions receive? Clearly, we should pass `*http.Request`,
and also, uh, _something_ for the field to extract. We can't explicitly type each argument here obviously;
so all we can do is pass the whole `args` struct's type, and that's what we do.

Next step is returning the function that extracts _all_ fields at once:

```go
    ...
	return func(r *http.Request) (args Args, err error) {
		for i := 0; err == nil && i < numFields; i++ {
			err = funcs[i](r, &args)
		}
		err = WithStatusCode(err, http.StatusBadRequest)
		return
	}
}
```

Next, our `extractFieldOfType` function should be able to operate on the given field of `Args`.

### Accessing the field

So far we have been fast forwarding similar parts. Now is the time to stop and think.

`argType.Field(i)` gave us `reflect.StructField`; which has the field's `reflect.Type`,
as well as the _memory offset_ from the base of `Args`. We want to end up with a reference to the field,
and cast that reference to the `Extractor` interface; so that we have some methods to call.

First things first, let's get the field's absolute memory address:

```go
func extractFieldOfType[Args any](fieldType reflect.Type, fieldOffset uintptr) func(*http.Request, *Args) error {
	return func(r *http.Request, argBase *Args) error {
        fieldPtr := unsafe.Add(unsafe.Pointer(argBase), fieldOffset)
        // ... ??
	}
}
```

Ok, now what? We have an `unsafe.Pointer` that points to the field to extract. If we knew the field's type as a
_generic_ type, like `T`, we could have done

```go
field := any((*T)(fieldPtr)).(Extractor)
```

and proceed from there.
But we don't know the field's type like this! All we have is `reflect.Type`!
How can we get an `any` from `unsafe.Pointer`?

Wait.

### NewAt

The `reflect` package _does_ have a gem for us.

```go
// in reflect/value.go

// NewAt returns a Value representing a pointer to a value of the
// specified type, using p as that pointer.
func NewAt(typ Type, p unsafe.Pointer) Value { ... }
```

We know that `fieldPtr` is pointing to a memory region that is `T`-sized for field of type `T`.
Why not point a `Value` for type `T` to this address? We can do the following:

```go
// pointer to the field
fieldPtr := unsafe.Add(unsafe.Pointer(argBase), fieldOffset)
// initialize field at address fieldPtr; get a reflect.Value; turn it into any
field := reflect.NewAt(fieldType, fieldPtr).Interface()
// cast to our interface and get to work
return field.(Extractor).Extract(r)
```

This has an __amazing__ side effect in terms of ergonomics:
In our previous implementation, our `Extractor`s were written a bit weirdly;
in that it had to work on a zero value to return a new result, wrapped into a `reflect.Value`.
They weren't like member functions, but had to behave like functions defined on the type itself.

__This time we are calling the member function of an _instantiated_ field__.
That means the method can directly modify its receiver value like normal Go code!

So our `Extractor` interface becomes very simply:

```go
type Extractor interface {
	Extract(*http.Request) error
}
```

## Generics out

Before we start implementing `Extractor`s... Since we didn't know how the function `extractFieldOfType` would shape,
we passed an `*Args` argument here. But this helper function does not have to be generic on `Args`; which would
increase compilation times and binary size. Instead, since we immediately cast `&args` into an `unsafe.Pointer`, we
can directly pass an `unsafe.Pointer` here.

In fact, even our `extractorFor` function does not have to be generic. We are already operating on the `reflect.Type`
of `Args`; for the one-time preparation; and for the handler we only need the `unsafe.Pointer`. Let's rewrite that
as well so that the little use of generics is in the very top-level `Handler` function.

Here is the entire code so far (except unchanged helper functions like `WriteResponse` from before):

```go
type Extractor interface {
	Extract(*http.Request) error
}

var extType = reflect.TypeFor[Extractor]()

const PanicReasonHandlerExpectsAStruct = "Handler argument should be a struct"
const PanicReasonUnknownFieldType = "Cannot determine how to extract handler argument field"

// Handler is generic because we want to strongly type the inner handler function
func Handler[Args, Output any](handler func(Args) (Output, error)) http.HandlerFunc {
    // Preparation is done using reflection ONLY
	extractInputs := extractorFor(reflect.TypeFor[Args]())
	return func(w http.ResponseWriter, r *http.Request) {
		var out any
		var args Args
		// extraction is done using pointers ONLY
		err := extractInputs(r, unsafe.Pointer(&args))
		if err == nil {
		    // here we have the typed Args
			out, err = handler(args)
		}
		if err == nil {
			_ = WriteResponse(w, out)
		} else {
			_ = writeErrResp(w, err)
		}
	}
}

func extractorFor(argType reflect.Type) func(*http.Request, unsafe.Pointer) error {
	if argType.Kind() != reflect.Struct {
		panic(PanicReasonHandlerExpectsAStruct)
	}
	numFields := argType.NumField()
	funcs := make([]func(*http.Request, unsafe.Pointer) error, numFields)
	for i := range numFields {
		field := argType.Field(i)
		if !reflect.PointerTo(field.Type).Implements(extType) {
			panic(PanicReasonUnknownFieldType)
		}
		funcs[i] = extractFieldOfType(field.Type, field.Offset)
	}
	// the returned function no longer takes Args, instead receives unsafe.Pointer
	return func(r *http.Request, argsPtr unsafe.Pointer) (err error) {
		for i, n := 0, len(funcs); err == nil && i < n; i++ {
			err = funcs[i](r, argsPtr)
		}
		err = WithStatusCode(err, http.StatusBadRequest)
		return
	}
}

func extractFieldOfType(
    fieldType reflect.Type, fieldOffset uintptr,
) func(*http.Request, unsafe.Pointer) error {
	return func(r *http.Request, argBasePtr unsafe.Pointer) error {
		fieldPtr := unsafe.Add(argBasePtr, fieldOffset)
		field := reflect.NewAt(fieldType, fieldPtr).Interface()
		return field.(Extractor).Extract(r)
	}
}
```

## Implementing the Extractors

Here come the fruits of our labor:

```go
type JSON[Inner any] struct{ V Inner }

func (v *JSON[T]) Extract(r *http.Request) error {
	return decodeBodyWith(r, json.NewDecoder, &v.V)
}
```

Compare that to the `JSON` implementation in the previous post:

```go
// cannot name the receiver here because it would be nil
func (*JSON[T]) Extract(r *http.Request) (any, error) {
	var v JSON[T] // must allocate and return a new JSON here
	return v, decodeBodyWith(r, json.NewDecoder, &v.V)
	// return type is any; if you return the wrong type
	// an angel loses its wings at runtime
}
```

The new one is more ergonomic, and free of pitfalls like this.

## Finalizing resources

There is one thing we didn't do: In our previous implementation, we gave `Extractors` a way to collect after them.
If the arguments of the function implemented `io.Closer`, we would call the `Close` functions at the end of handler.
We should do that here as well.

First, let's implement the `Logger` extractor so that we can see a use case:

```go
type Logger struct{ *slog.Logger }

func (l *Logger) Extract(r *http.Request) error {
	l.Logger = slog.Default().With("route", r.URL.Path, "request_id", uuid.NewString())
	l.Info("endpoint start")
	return nil
}

func (l *Logger) Close() error {
	l.Logger.Info("endpoint end")
	return nil
}
```

If we run our previous 'test', we won't see the "endpoint end" log at the end of the handler:

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

The output:

```
2024/12/28 01:27:52 INFO endpoint start route=/users request_id=56601648-63ce-41db-8558-170c03440569
2024/12/28 01:27:52 INFO created route=/users request_id=56601648-63ce-41db-8558-170c03440569 user=abc
201: {"id":1337,"username":"abc"}

2024/12/28 01:27:52 INFO endpoint start route=/users request_id=7dd0031d-c7d5-41f1-96e8-2ed509e554bb
400: {"error":"invalid character '{' looking for beginning of object key string"}
```

First of all; we want to check if each extracted field also implements `io.Closer`.
That means `extractFieldOfType` also returns this interface as well:

```go
func extractFieldOfType(
	fieldType reflect.Type, fieldOffset uintptr,
) func(*http.Request, unsafe.Pointer) (io.Closer, error) {
	return func(r *http.Request, argBasePtr unsafe.Pointer) (io.Closer, error) {
		fieldPtr := unsafe.Add(argBasePtr, fieldOffset)
		field := reflect.NewAt(fieldType, fieldPtr).Interface()
		// if field is also an io.Closer, cast that as well.
		// if the cast fails; this will return nil anyway
		closer, _ := field.(io.Closer)
		return closer, field.(Extractor).Extract(r)
	}
}
```

Since multiple fields can return as closers, we want to collect them so that we can call them all at once:

```go
type closers []io.Closer

func (c closers) Close() (err error) {
    // iterate in reverse similar to defers
	for i := len(c) - 1; i >= 0; i-- {
	    // this is cleanup; don't stop at the first error
		err = errors.Join(err, c[i].Close())
	}
	return
}
```

Then we use it in `extractorFor` like this:

```go
func extractorFor(argType reflect.Type) func(*http.Request, unsafe.Pointer) (io.Closer, error) {
	...
	// funcs now also return an optional io.Closer
	funcs := make([]func(*http.Request, unsafe.Pointer) (io.Closer, error), numFields)
	for i := range numFields {
	    ...
		funcs[i] = extractFieldOfType(field.Type, field.Offset)
	}
	// since closer array also implements io.Closer; return it as one io.Closer
	return func(r *http.Request, argsPtr unsafe.Pointer) (io.Closer, error) {
		c := make(closers, 0, numFields)
		for _, f := range funcs {
			if closer, err := f(r, argsPtr); err != nil {
				return c, WithStatusCode(err, http.StatusBadRequest)
			} else if closer != nil {
			    // collect all non-nil closers in the array
				c = append(c, closer)
			}
		}
		return c, nil
	}
}
```

Finally, we call this cleanup operation in the `Handler` after the endpoint is handled:

```go
func Handler[Args, Output any](handler func(Args) (Output, error)) http.HandlerFunc {
	extractInputs := extractorFor(reflect.TypeFor[Args]())
	return func(w http.ResponseWriter, r *http.Request) {
		var out any
		var args Args
		// cleanup operations are also returned here
		cleanup, err := extractInputs(r, unsafe.Pointer(&args))
		if err == nil {
			out, err = handler(args)
		}
		// cleanup here. doing this in a defer might be safer btw
		err = errors.Join(err, cleanup.Close())
        ...
	}
}
```

After this, we see the "endpoint end" logs as well:

```
2024/12/28 01:42:09 INFO endpoint start route=/users request_id=6d481ca6-1917-46dc-9a6b-23b5e07398d3
2024/12/28 01:42:09 INFO created route=/users request_id=6d481ca6-1917-46dc-9a6b-23b5e07398d3 user=abc
2024/12/28 01:42:09 INFO endpoint end route=/users request_id=6d481ca6-1917-46dc-9a6b-23b5e07398d3
201: {"id":1337,"username":"abc"}

2024/12/28 01:42:09 INFO endpoint start route=/users request_id=cdf82e94-2f44-44ee-8a9a-e12d2cc85e76
2024/12/28 01:42:09 INFO endpoint end route=/users request_id=cdf82e94-2f44-44ee-8a9a-e12d2cc85e76
400: {"error":"invalid character '{' looking for beginning of object key string"}
```

## Final notes

- Overall we ended up with a cleaner, safer and more performant design that is more readable in my opinion as well.
- One change I did not mention here is the output type. In our example endpoint, we are now returning `User` directly
  instead of `JSON[User]`. I was trying to closely imitate the axum API in the first post; but in general defaulting to
  JSON serialization handles most cases anyway. If you want to return something else for an endpoint, simply implement
  `Responder` on the return type. Writing this kind of logic in wrapper types instead complicates things for no reason.
- One huge caveat in this implementation is _embedding_ structs that contain `Extractors`. Remember that our
  `extractorFor` function only iterates over the top-level fields; so if your function parameters look like this:

```go
type CreateUserParams struct {
    Common
    JSON[struct {...}]
}

type Common struct {
    Logger
    UserStore
}

func createUser(p CreateUserParams) (User, error) { ... }
```

Then the extractors in `Common` will not be resolved. The solution to this is largely out of scope for this blog post,
but you can either:

1. Recurse into inner structs that don't implement `Extractor`, adding each level's field offset as well
2. Find a way to implement `Extractor` for `Common`, and delegate to inner fields

- The order in which parameters are extracted is dependent on the order of fields in the struct.
  This was also the case for our previous implementation. If somehow the order of these operations are important,
  then instead of _returning_ extractor functions you can pass a registry of functions where each extractor adds
  itself. Then you add stages to the registry; and let each extractor also specify the order. That way extractors
  can dictate the order of operations. I don't think this has many use cases.

- The fields of the input struct are embedded in all examples, this is not a requirement. The version below works
  equally well:

```go
func createUser(p struct {
	log Logger
	body JSON[CreateUser]
}) (User, error) {
	p.log.Info("created", "user", p.body.V.Username)
	return User{ID: 1337, Username: p.body.V.Username}, nil
}
```

It is just more verbose.

- If you think that extractors may need other parameters than `*http.Request` for things like state, database etc.
  you can add them to the interface. Or just (ab)use the request's `Context` to pass such things into the function.
  I generally prefer the latter, using the `Context` for dependency injection lets you mock pretty much any external
  dependency.

- By the way, by tweaking the 'closer' logic a little, you can manage even database transactions through `Extractor`s.
  We piggybacked on the `io.Closer` interface, but if you write your own finisher interface that has two functions
  ("Run this one in case of success", "Run this one in any situation") you can implement commit-rollback operations
  in them.

That’s all I have for this post. Go now.