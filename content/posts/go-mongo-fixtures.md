+++
title = "Implementing MongoDB Test Fixtures in Go"
date = "2025-06-17T03:50:10+03:00"
keywords = ['go', 'mongodb', 'test']
description = """Here is a way to load test fixtures for MongoDB in Go."""
showFullContent = false
readingTime = false
hideComments = true
toc = true
+++

## Introduction

At some point in the recent past; I needed to port the integration tests of a Go service using PostgreSQL 
to MongoDB. I was surprised by the lack of resources for loading test fixtures, so I decided to document
my approach here.

For PostgreSQL, we use [go-testfixtures](https://github.com/go-testfixtures/testfixtures)
to describe our test data as `.yml` files, which (naturally) does not support MongoDB.
I wanted a similar approach for MongoDB.


### Test Fixtures

In Ruby on Rails, applications with database access are tested using 
[test fixtures](https://guides.rubyonrails.org/testing.html#fixtures) described in YAML files.
Before the test, each table is populated with the data described in the corresponding YAML file:

```yaml
# test/fixtures/users.yml
david:
  name: David Heinemeier Hansson
  birthday: 1979-10-15
  profession: Systems development

steve:
  name: Steve Ross Kellock
  birthday: 1974-09-27
  profession: guy with keyboard
```

This fixture will put two rows in the `users` table.

`go-testfixtures` applies the same approach to Go:

```go
fixtures, err := testfixtures.New(
        testfixtures.Database(db),
        testfixtures.Dialect("postgres"),
        testfixtures.Paths(
                "fixtures/orders.yml",
                "fixtures/customers.yml",
                "common_fixtures/users"
        ),
)
```

This will load the two files from the `fixtures/` directory, as well as any `.yml` files under the `users/` directory. 
And this can be very useful for integration tests!

> I want to point out two things here.
> 
> First: one footgun with this approach is that the library **wipes the entire database before 
> loading the fixture**, so you need to make sure you're connected to the correct database. 
> This is easily solvable with the `testfixtures.DangerousSkipTestDatabaseCheck()` option, which
> ensures that the database name contains `test`.
> 
> Secondly; if you're running queries against an actual database, you cannot parallelize DB-facing tests.
> This is not specific to `go-testfixtures` and most organizations are fine with this situation; but if 
> you really need parallel testing, I've seen the [begin-query-rollback trick](https://github.com/DATA-DOG/go-txdb) 
> gain some traction lately.

`go-testfixtures` also supports templates, but we don't use this feature. What we _do_ use is this though:

```yaml
- id: 1
  created_at: now() # or, more explicitly, RAW=NOW()
  updated_at: now()
  # ...
```

PostgreSQL sees this and evaluates the `now()` query as the current timestamp. Passing raw queries like 
this is super useful especially for things like timestamps that need not have a fixed value for the test.
I'd like the same feature in my MongoDB fixture loader.

### MongoDB

MongoDB internally uses a binary format named BSON, but for compatibility reasons a JSON extension named 
[Extended JSON](https://www.mongodb.com/docs/manual/reference/mongodb-extended-json/) is also supported.

Extended JSON is valid JSON; but it also preserves things like type information more precisely.
`mongodb` tools like `mongoexport` all support Extended JSON.

> Now; you can directly 
> [import Extended JSON documents](https://www.mongodb.com/resources/languages/json-to-mongodb)
> into MongoDB using `mongoimport`, which is available in the `mongo-tools` package. 
> But since loading test fixtures is a simple job, and relying on the existence of an 
> external CLI tool, and spawning it for every test seems to be a little brittle to me.

Cool. So, each collection will be a separate file. Since each collection can have multiple 
documents, we should be able to represent multiple records. I can store the array of records
as a JSON array; but since JSON disallows trailing commas it would be slightly harder to keep tidy:

```json
[
  {"name": "david"},
  {"name": "steve"},   <- syntax error here, due to the extra comma :(
]
```

Instead, we can use the [JSON Lines](https://jsonlines.org/) spec. It is basically multiple JSON
documents separated by a newline:

```json lines
{"name": "david"}
{"name": "steve"}
```

I'm using JetBrains Goland as IDE, which has support for this if I set the extension to `.jsonl`.
Also, since Extended JSON is syntactically valid JSON, it is compatible with JSONL.

### Walking the Directories

Let's get cooking then. First, let's grab the mongo driver library:

```shell
go get go.mongodb.org/mongo-driver
```

This is the only dependency we will need. Our `LoadMongoFixtures` function will receive a ref to the 
`Database`, and a list of paths to load fixtures from:

```go
func LoadMongoFixtures(ctx context.Context, db *mongo.Database, paths ...string) error {
    // identify valid fixture files in the given paths
	files, err := walkPaths(paths)
	if err != nil {
		return fmt.Errorf("cannot walk paths: %w", err)
	}
	// for each file; load the fixture
	for _, f := range files {
		if err = loadOneMongoFixture(ctx, db, f.filePath, f.collection); err != nil {
			return err
		}
	}
	return nil
}

type pair struct{ filePath, collection string }

func walkPaths(paths []string) ([]pair, error) { ... }

func loadOneMongoFixture(
    ctx context.Context, db *mongo.Database, filePath, collection string,
) error { ... }
```

`walkPaths` returns pairs of file path and collection name strings, which we iterate on and load one by one:

```go
func walkPaths(paths []string) ([]pair, error) {
	var files []pair
	for _, filePath := range paths {
		// stat this path first to see if it's a file or a directory
		pathInfo, err := os.Stat(filePath)
		if err != nil {
			return nil, fmt.Errorf("cannot stat file %q: %w", filePath, err)
		}
		// for files; directly append this as an entry
		if !pathInfo.IsDir() {
			// if the file is explicitly added; don't check the extension
			files = append(files, pair{
				filePath:   filePath,
				collection: strings.SplitN(filepath.Base(filePath), ".", 2)[0],
			})
			continue
		}
		// for directories; we should read and iterate them
		dirInfo, err := os.ReadDir(filePath)
		if err != nil {
			return nil, fmt.Errorf("cannot stat dir %q: %w", filePath, err)
		}
		files = walkDirs(filePath, dirInfo, files)
	}
	return files, nil
}
```
Nothing too fancy; if the path is a single file; we don't check the extension because it was explicitly added.
If it is a directory, we walk the dir and load files in it using `walkDirs`, which goes like this:

```go
func walkDirs(p string, dir []os.DirEntry, files []pair) []pair {
	for _, file := range dir {
		// skip inner dirs
		if file.IsDir() {
			continue
		}
		// only process json / jsonl files
		name := file.Name()
		if ext := filepath.Ext(name); ext != ".jsonl" && ext != ".json" {
			continue
		}
		// append the file
		files = append(files, pair{
			filePath:   path.Join(p, name),
			collection: strings.SplitN(name, ".", 2)[0],
		})
	}
	return files
}
```

This one also checks for the file extension.

### Loading the Fixtures

After this we actually load each fixture file using `loadOneMongoFixture`.
The first order of business is to open the file:

```go
func loadOneMongoFixture(ctx context.Context, db *mongo.Database, filePath, collection string) error {
	// open the file for streaming
	file, err := os.Open(filePath)
	if err != nil {
		return fmt.Errorf("cannot open file %s: %w", filePath, err)
	}
	defer func() { _ = file.Close() }()
	...
	return nil
}
```
Then we clear the collection:
```go
	// clear the collection first
	if _, err = db.Collection(collection).DeleteMany(ctx, bson.M{}); err != nil {
		return fmt.Errorf("cannot clear collection %s: %w", collection, err)
	}
```

OK, time to read some JSON lines! Unlike `json.Unmarshal`, `json.NewDecoder` allows trailing 
tokens in the stream unless explicitly checked against; so we can simply loop with it until `.More()` returns false.
This is better than splitting from newlines; because we actually don't want _lines_ of JSON:
A JSON record can have newlines in it due to formatting. What we actually want is JSON after JSON, basically:

```go
	// stream the file
	for dec := json.NewDecoder(file); dec.More(); { ... }
}
```

Now is the time to slow down because the boilerplate is over: we are getting into the business logic here.

### EJSON to BSON

Inside that loop, we should first read one record of Extended JSON, then convert it to BSON.

Since EJSON is syntactically valid JSON; we can use the JSON decoder to obtain the boundaries of the message like this:

```go
		// first, decode into a RawMessage. this is necessary only to identify the boundaries of one entry
		var raw json.RawMessage
		if err = dec.Decode(&raw); err != nil {
			return fmt.Errorf("cannot decode fixture %s: %w", collection, err)
		}
```

This reads one JSON record worth of data into a `json.RawMessage` chunk, which is `[]byte` basically.
At this point, we had split off one record from an array of EJSON records. Now, we can pass this slice 
of bytes to `bson.UnmarshalExtJSON` to obtain a `bson.M` map object, which is the actual unmarshalling step:

```go		
		// now; using the raw message as the input; parse the json into a bson map
		var doc bson.M
		if err = bson.UnmarshalExtJSON(raw, false, &doc); err != nil {
			return fmt.Errorf("cannot unmarshal fixture %s: %w", collection, err)
		}
```

MongoDB here we come!

```go
		if _, err = db.Collection(collection).InsertOne(ctx, doc); err != nil {
			return fmt.Errorf("cannot insert document to collection %s: %w", collection, err)
		}
```

### Testing it

We need to test this. First, some helpers:

```go
var testDB *mongo.Database

func TestMain(m *testing.M) {
	cli, err := mongo.Connect(context.Background(),
		options.Client().ApplyURI("mongodb://localhost:27017"))
	if err != nil {
		log.Fatal(err.Error())
	}
	testDB = cli.Database("example")
	os.Exit(m.Run())
}

func assertEq(t *testing.T, expected, actual any) {
	if t.Helper(); expected != actual {
		t.Fatalf("assertion failed (expected: %v, actual: %v)", expected, actual)
	}
}
```

`TestMain` sets up our database connection. I hardcoded the database config because 
this is a blog post, cut me some slack here ok? I also threw in `assertEq` for shorter code later.

Let's write a test:

```go
type Post struct {
	Title       string    `bson:"title"`
	PublishedAt time.Time `bson:"published_at"`
}

func TestLoadFixtures(t *testing.T) {
	ctx := t.Context()
	err := LoadMongoFixtures(ctx, testDB, "fixtures/")
	assertEq(t, nil, err)
	c, err := testDB.Collection("posts").Find(ctx, bson.D{})
	assertEq(t, nil, err)
	var records []Post
	assertEq(t, nil, c.All(ctx, &records))
	assertEq(t, 2, len(records))
}
```
Ok, we simply load the mongo fixtures under the `fixtures/` path; then read all entries
in the `posts` collection and decode them in an array of `Post`s.

Here is our test fixture for this test; `fixtures/posts.jsonl`:

```json lines
{
  "title": "Hello world!",
  "published_at": {
    "$dateSubtract": {
      "startDate": "$$NOW",
      "unit": "minute",
      "amount": 10
    }
  }
}
{
  "title": "Hello again!",
  "published_at": {
    "$dateSubtract": {
      "startDate": "$$NOW",
      "unit": "minute",
      "amount": 5
    }
  }
}
```

All we need to do is to spin up a local Mongo instance and run the test.

See how we wrote those "now minus 5 minutes" kind of timestamps? Isn't that cool that MongoDB supports th-

```
=== RUN   TestLoadFixtures
    fixture_test.go:44: assertion failed (expected: <nil>, actual: error decoding key published_at: cannot decode embedded document into a time.Time)
--- FAIL: TestLoadFixtures (0.01s)
```

### The hack (or, the reason I wrote a blog post)

Ok ummm I may have celebrated prematurely. Apparently, Mongo Compass evaluates this, so does `mongoimport`; but
when you use the insert flow it is treated as a dictionary as-is.

Fortunately, there is a catch! `UpdateOne` actually _does_ evaluate expressions in the updated 
documents when passed a pipeline. This allows us to pass things like dynamic timestamps using `$currentDate`.

However, we want to _insert_ documents, not _update_ them. Well, how about we do an upsert,
provide an impossible-to-match filter to it, so it never matches, therefore it always behaves as an insert.

Since all documents in Mongo have an `_id` field, we can check for its absence.
```go
		// replacing the InsertOne with this
		if _, err = db.Collection(collection).UpdateOne(ctx,
			// "update the documents without an _id field"
			bson.M{"_id": bson.M{"$exists": false}},
			// this has to be an array for it to be considered a pipeline
			[]bson.M{{"$set": doc}},
			// setUpsert lets us abuse this to behave as an insert
			options.Update().SetUpsert(true),
		); err != nil {
			return fmt.Errorf("cannot insert document to collection %s: %w", collection, err)
		}
```

When we run it again the test passes!

```
=== RUN   TestLoadFixtures
--- PASS: TestLoadFixtures (0.01s)
```

If we add extra assertions, we see that the timestamps are correctly evaluated.

That's all I have for tonight. Go now.