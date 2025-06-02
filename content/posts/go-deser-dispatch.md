+++
title = "Automatic Go union deser dispatch with tags"
date = "2025-06-02T00:27:23+03:00"
keywords = ['axum', 'go', 'http']
description = """Here is another quick use case for the field offset trick we pulled in the previous posts."""
showFullContent = false
readingTime = false
hideComments = true
toc = true
+++

## Introduction

Ok, this will be a short one.

In the [previous posts](/posts/go-axum-handlers-pt2/); we tried to reduce endpoint handler boilerplate in Go 
by imitating the Axum magic functions. This post is ***not*** about that, but I think one of the tricks we 
pulled there may have another use case.

If you'll recall, we iterated through two separate solutions to the endpoint problem, where in the latter one 
we used struct fields to represent endpoint parameters.
To initialize those fields into `any` references that internally retain type info, we used `reflect.NewAt` 
with the field offset like this:

```go
fieldPtr := unsafe.Add(unsafe.Pointer(structPtr), fieldOffset)
fieldRef := reflect.NewAt(fieldType, fieldPtr).Interface()
```

Since we get `fieldOffset` and `fieldType` from a `reflect.StructField` that we obtained through iterating the struct's
type reflectively; the offset into the struct is guaranteed to hold the field's type. This will be useful.

Enough recap. If you didn't read that post, you don't really have to. I'll go over this approach in this post anyway.

## The problem

This time, we are trying to solve another problem.

I am currently dealing with a JSON API that returns lists of different kinds of items.
The API awkwardly has a single endpoint, where the type of the returned `items` array is 
discerned using a tag field (`type`). Here is what I mean.

Let's say that the API can return an array of either `Crustacean`s or `Rodent`s:

```go
type Crustacean struct {
    HasCarapace bool   `json:"has_carapace"`
    Color       string `json:"color"`
}

type Rodent struct {
    IsDigging bool `json:"is_digging"`
    NumTeeth  uint `json:"num_teeth"`
}
```

The response is either:

```json
{
  "type": "gopher",
  "items": [
    {"is_digging": true, "num_teeth": 4},
    ...
  ]
}
```
or:
```json
{
  "type": "crab",
  "items": [
    {"has_carapace": true, "color": "red"},
    ...
  ]
}
```

## Enter boilerplate

The first issue is to represent this with a Go struct. Since we don't really have union types in Go, 
we have to awkwardly write a sum type to handle all possible responses:

```go
type Items struct {
    ItemType    string

    // Depending on the item type; only one of
    // these arrays is potentially non-empty.
    Crustaceans []Crustacean
    Rodents     []Rodent
}
```

Since all item types are returned in the same `items` field; we must write a custom unmarshal function:
```go
func (items *Items) UnmarshalJSON(b []byte) (err error) {
    // inlined data type to deserialize the type field.
    // also extract the items array's bytes; but as raw msg
    var t struct {
        Type  string          `json:"type"`
        Items json.RawMessage `json:"items"`
    }
    // first deserialize the general shape of the message.
    // this will give us the message type field. use it to
    // dispatch to separate unmarshal calls.
    if err = json.Unmarshal(b, &items); err == nil {
        // set the item type field, then dispatch
        items.ItemType = t.Type
        switch t.Type {
        case "crab":
            // type: crab unmarshals items into the Crustaceans array
            err = json.Unmarshal(t.Items, &items.Crustaceans)
        case "gopher":
            // type: gopher unmarshals items into the Rodents array
            err = json.Unmarshal(t.Items, &items.Rodents)
        default:
            err = errors.New("unknown item type " + t.Type)
        }
    }
    return
}
```

I hope this is straightforward:
1. First we parse the json to identify the `type` field. During this, we also get the value of the `items` key as
a raw byte array using `json.RawMessage`.
2. Using the value of the `type` field, we employ a switch-case to deserialize this `RawMessage` differently.
If `type=="crab"`, we treat the bytes as `[]Crustacean`, if `type=="gopher"` we deserialize them as `[]Rodent`.

This does work; but there is something about that switch-case that bothers me. In the actual scenario, I'm dealing with
more than 10 item types; so the switch-case becomes quite large.

Also, consider what we should do to add a new item type here, like a `Camel`:
1. Add a new `Camels []Camel` field to the `Items` struct.
2. Add a new `case "camel":` case to the unmarshal code.
3. While copy-pasting code, make sure you change the `Unmarshal` target to `&items.Camels`.

Omitting step #2, or making a mistake in #3 will not cause compilation errors. In the worst possible case;
two subtypes can be close enough that the actual deserialization also does not fail; causing the wrong logic 
to be executed eventually.

### A small aside
Also consider the usage of this type. We end up with another switch-case while using it:
```go
func VisitItems(items *Items) {
    switch items.ItemType {
    case "crab":
        for _, e := range items.Crustaceans {
            handleCrustacean(e)
        }
    case "gopher":
        for _, e := range items.Rodents {
            handleRodent(e)
        }
    }
}
```
Although we can do a little evil here and abuse the fact that only one of these arrays have elements, like this:
```go
func VisitItems(items *Items) {
    for _, e := range items.Crustaceans {
        handleCrustacean(e)
    }
    for _, e := range items.Rodents {
        handleRodent(e)
    }
}
```
This way we won't actually need to have an `ItemType` field in `Items`. I'm not advocating you to do this, though;
depending on your case the latter may be significantly less readable and/or brittle.

Anyway, where were we?

## Using reflection

The problem is that we are making highly coupled changes in two locations at once.

Wouldn't it be (slightly) better if we could just do this?

```go
type Items struct {
    Crustaceans []Crustacean `item:"crab"`
    Rodents     []Rodent     `item:"gopher"`
}

func (items *Items) UnmarshalJSON(b []byte) (err error) {
    var t struct {
        Type  string          `json:"type"`
        Items json.RawMessage `json:"items"`
    }
    if err = json.Unmarshal(b, &t); err == nil {
        // which field should we read into? determine using t.Type
        if fieldInfo, isKnownType := itemFields[t.Type]; isKnownType {
            // get a reference to the relevant field as `any`, and unmarshal into it
            err = json.Unmarshal(t.Items, fieldInfo.getRef(items))
        } else {
            err = errors.New("unknown item type " + t.Type)
        }
    }
    return
}
```

Note how we associated each field's type name with it using a struct tag named `item`.
We can populate the `itemFields` map using a little reflection as follows:

```go
// populate this only once during module load
var itemFields = structFieldGetter[Items]("item")

func structFieldGetter[T any](tagName string) map[string]FieldInfo[T] {
	fields := make(map[string]FieldInfo[T])
	// first, obtain type for the struct
	structType := reflect.TypeFor[T]()
	// iterate all struct fields
	for idx := range structType.NumField() {
		structField := structType.Field(idx)
		// if the field has the given tag
		if fieldName := structField.Tag.Get(tagName); fieldName != "" {
			// save its offset and type
			fields[fieldName] = FieldInfo[T]{
				fieldOffset: structField.Offset,
				fieldType:   structField.Type,
			}
		}
	}
	return fields
}

type FieldInfo[T any] struct {
	// 0-sized phantom data for type safety
	_ [0]*T

	fieldOffset uintptr
	fieldType   reflect.Type
}

// get a reference to the field of the struct
func (f FieldInfo[T]) getRef(structRef *T) any {
    // todo
}
```

Ok, we create a map from the specified item type string to a `FieldInfo` struct.
`FieldInfo` records the type and offset of each eligible field, which we will later use to get 
a reference to that field when we have an `Items` struct.

> The "0-sized phantom data" there is needed, because without that `FieldInfo[T]` and `FieldInfo[Y]` 
> would have the same layout for different `T` and `Y`, and the compiler wouldn't prevent casts from 
> one to the other. We need that `T` to not change so that we can guarantee `getRef`'s type safety.

## Here it comes

To implement `getRef`, we will use the trick we described earlier.

```go
func (f FieldInfo[T]) getRef(structRef *T) any {
	// convert to unsafe pointer
	structPtr := unsafe.Pointer(structRef)
	// trivial pointer arithmetic to get the field's address
	fieldPtr := unsafe.Add(structPtr, f.fieldOffset)
	// instantiate the field type at the given address; return as an interface
	return reflect.NewAt(f.fieldType, fieldPtr).Interface()
}
```

Since we obtained the field info by iterating `T`, and also received a `*T` in `getRef`; we ensure that the 
given offset will have the given type. Then we can safely use `reflect.NewAt` to instantiate the field at the
given address. This gives us a valid `reflect.Value`, which we then can convert to `any` using `.Interface()`.
Then the unmarshaller code can use it as a destination for the `items` array's bytes. Voila!

Once again, the reflection code that walks the struct's fields runs only once at the start of the program.
Some of the reflection does take place during `getRef`, but `json.Unmarshal` already uses reflection 
internally, so it's not like we are introducing RTTI where it wasn't previously needed.

With this, adding a new item type requires updates in only one site:

```go
type Items struct {
    Crustaceans []Crustacean `item:"crab"`
    Rodents     []Rodent     `item:"gopher"`
    Camels      []Camel      `item:"camel"` // <----
}
```

And the unmarshaller will automatically start to handle camels as well.

## There's more

This approach also has another (albeit minor) advantage.

We could have simply put `getRef` directly into the map as an anonymous function, but we chose to create 
a `FieldInfo` type where `getRef` was a member function. This is because fields can carry more info, which 
we can now store in the `FieldInfo` struct!

I'll give you an example.

Remember the API I was reading from? Well, that API has pagination. Since each item can have wildly different sizes,
it makes sense to have a different page size for different item types. For example, I can easily fetch 1000 
crustacean rows at once, but camels have a "comment" field that have paragraphs of text; so maybe I shouldn't
exceed 100 entries in one page.

Let's model that:

```go
type Items struct {
    Crustaceans []Crustacean `item:"crab,page_size=1000"`
    Rodents     []Rodent     `item:"gopher,page_size=1000"`
    Camels      []Camel      `item:"camel,page_size=100"`
}
```

Now all we have to do is parse the tag text to obtain this field. I'll omit that code because it's 
trivial and beside the point; but as a result we add an `PageSize` field to the `FieldInfo` struct:

```go
type FieldInfo[T any] struct {
	_ [0]*T

	fieldOffset uintptr
	fieldType   reflect.Type

	PageSize uint
}
```

> If you'd like to keep this struct general-purpose, maybe an `Attr map[string]any` field may be better.
> It's late here, so instead I want to get to the point and go to sleep.

Then we can get the page size for each item type as follows:

```go
pageSize := itemFields[itemType].PageSize
```

The alternative would have been keeping a separate map like this:

```go
var pageSizes = map[string]uint{
    "crab": 1000, "gopher": 1000, "camel": 100, ...
}

...

pageSize := pageSizes[itemType]
```
which we would have to maintain and make sure we didn't miss any of the keys.

Anyway, this is all I have for today. Go now.
