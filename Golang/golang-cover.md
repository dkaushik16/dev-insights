# Go for MERN Developers — Zero to Interview-Ready

A complete Go (Golang) tutorial written specifically for someone who already thinks in JavaScript/Node/React/Express/MongoDB terms. Every new concept is anchored to something you already know.

---

## Table of Contents

1. [Mental Model: Go vs Node](#1-mental-model-go-vs-node)
2. [Setup & Tooling](#2-setup--tooling)
3. [Syntax Basics: Variables, Types, Constants](#3-syntax-basics)
4. [Control Flow](#4-control-flow)
5. [Functions](#5-functions)
6. [Arrays, Slices, Maps](#6-arrays-slices-maps)
7. [Structs & Methods (Go's "classes")](#7-structs--methods)
8. [Interfaces (Go's duck typing)](#8-interfaces)
9. [Error Handling (no try/catch)](#9-error-handling)
10. [Pointers](#10-pointers)
11. [Packages & Modules (Go's npm)](#11-packages--modules)
12. [Goroutines & Channels — Go's Superpower](#12-goroutines--channels)
13. [sync: WaitGroup, Mutex](#13-sync-package)
14. [Context (cancellation, like AbortController)](#14-context)
15. [Generics](#15-generics)
16. [Testing](#16-testing)
17. [Building a REST API](#17-building-a-rest-api)
18. [Databases: database/sql, GORM, MongoDB](#18-databases)
19. [Project Structure & Best Practices](#19-project-structure)
20. [Interview Cheat Sheet](#20-interview-cheat-sheet)
21. [What to Build Next](#21-what-to-build-next)

---

## 1. Mental Model: Go vs Node

| Concept | Node/JS/Express | Go |
|---|---|---|
| Runtime | V8 engine, interpreted/JIT | Compiled to a single native binary |
| Concurrency | Event loop, single-threaded, async/await | Goroutines (green threads), true parallelism across CPU cores |
| Typing | Dynamic (or TS on top) | Static, compiled — errors caught before running |
| Package manager | npm/yarn + package.json | `go mod` + go.mod |
| Async pattern | Promises, async/await, callbacks | Goroutines + channels (CSP model) |
| Error handling | try/catch, throw | Explicit `error` return values, no exceptions for normal flow |
| Web framework | Express, Fastify | net/http (stdlib is already production-grade), or Gin/Chi/Fiber |
| ORM | Mongoose, Prisma, Sequelize | GORM, sqlc, or raw database/sql |
| Class-like OOP | `class`, prototypal inheritance | No classes — structs + methods + interfaces (composition over inheritance) |
| Null | `null`/`undefined` | `nil` (zero value for pointers, slices, maps, interfaces, channels, funcs) |
| Deployment | Needs Node runtime installed | Ship one static binary, no runtime needed |

**Biggest mindset shift:** In Node you reach for `async/await` and `Promise.all`. In Go you reach for `go func(){}()` (goroutine) and `channels`. Both solve concurrency, but Go's model gives you real parallelism (multi-core) with much lower overhead — this is why Go dominates backend infra, CLIs, and microservices.

---

## 2. Setup & Tooling

```bash
# Install (macOS)
brew install go

# Verify
go version

# Create a project (like npm init)
mkdir myapp && cd myapp
go mod init github.com/yourname/myapp   # creates go.mod (like package.json)

# Run a file directly (like `node index.js`)
go run main.go

# Build a compiled binary (like a production bundle, but it's an actual executable)
go build -o myapp

# Format code (built-in, non-negotiable in Go community — like Prettier but standardized)
gofmt -w .
# or the newer, smarter version:
go vet ./...
```

Key files:
- `go.mod` = `package.json` (module name, Go version, dependencies)
- `go.sum` = `package-lock.json` (checksums for reproducible builds)

There's no `node_modules` — dependencies are cached globally in `$GOPATH/pkg/mod` and referenced by version, not copied per-project.

---

## 3. Syntax Basics

### Variables

```go
package main

import "fmt"

func main() {
    // Explicit type
    var name string = "Alice"

    // Type inference (most common — like `let` but Go infers the type once, forever)
    age := 25

    // Multiple declarations
    var (
        x int    = 10
        y string = "hello"
    )

    // Constants (like `const` in JS, but compile-time and can be untyped)
    const Pi = 3.14159

    fmt.Println(name, age, x, y, Pi)
}
```

Key differences from JS:
- `:=` is short variable declaration — only usable inside functions, infers type. Once a variable's type is set, it's fixed forever (no reassigning a string to a number).
- Every declared variable **must be used**, or the compiler errors. No unused imports either. This feels strict at first, keeps codebases clean forever.
- No `var`/`let` distinction — Go has block scoping like `let` by default.

### Basic Types

```go
// Numbers - Go is explicit about size (unlike JS's single `number` type)
var i int        // platform-dependent, usually 64-bit
var i8 int8       // -128 to 127
var i32 int32
var i64 int64
var u uint        // unsigned
var f32 float32
var f64 float64   // most common for decimals

// Strings - immutable, UTF-8 by default
var s string = "hello"

// Booleans
var b bool = true

// Zero values — Go has NO `undefined`. Every type has a default "zero value":
var zi int        // 0
var zs string      // ""
var zb bool        // false
var zp *int        // nil
```

This is a big one: **there is no `undefined`**. An uninitialized `int` is `0`, an uninitialized `string` is `""`, an uninitialized pointer/slice/map/interface is `nil`. This eliminates a whole class of JS bugs but means you must always know a variable's zero value.

### Type Conversion (explicit, no coercion)

```go
var i int = 42
var f float64 = float64(i)   // must convert explicitly
var s string = fmt.Sprintf("%d", i) // or strconv.Itoa(i)

// JS does `"5" + 5 = "55"` silently. Go will not compile that. This is a feature.
```

---

## 4. Control Flow

### If/Else — no parentheses, braces mandatory

```go
age := 20
if age >= 18 {
    fmt.Println("adult")
} else if age >= 13 {
    fmt.Println("teen")
} else {
    fmt.Println("child")
}

// If with an init statement (very common in Go, especially with errors)
if val, ok := someMap["key"]; ok {
    fmt.Println(val)
}
```

### Loops — Go has only `for` (no `while`, no `do-while`, no `forEach` keyword)

```go
// classic for
for i := 0; i < 5; i++ {
    fmt.Println(i)
}

// while-style
n := 0
for n < 5 {
    n++
}

// infinite loop (like `while(true)`)
for {
    break
}

// range — like `for...of` / `.forEach`
nums := []int{10, 20, 30}
for index, value := range nums {
    fmt.Println(index, value)
}

// range over a map — like Object.entries()
m := map[string]int{"a": 1, "b": 2}
for key, val := range m {
    fmt.Println(key, val)
}
```

### Switch (no fallthrough by default — opposite of JS!)

```go
switch day := "Mon"; day {
case "Sat", "Sun":
    fmt.Println("weekend")
case "Mon":
    fmt.Println("start of week")
    // no `break` needed, Go breaks automatically
default:
    fmt.Println("weekday")
}
```

---

## 5. Functions

```go
// Basic function
func add(a int, b int) int {
    return a + b
}

// Shorthand when params share a type
func add(a, b int) int {
    return a + b
}

// Multiple return values — used EVERYWHERE in Go, especially for errors
func divide(a, b int) (int, error) {
    if b == 0 {
        return 0, fmt.Errorf("cannot divide by zero")
    }
    return a / b, nil
}

result, err := divide(10, 2)
if err != nil {
    fmt.Println("error:", err)
}

// Named return values
func split(sum int) (x, y int) {
    x = sum * 4 / 9
    y = sum - x
    return // "naked" return — returns x, y automatically
}

// Variadic functions — like JS rest params (...args)
func sum(nums ...int) int {
    total := 0
    for _, n := range nums {
        total += n
    }
    return total
}
sum(1, 2, 3) // 6

// Functions as values / closures — same as JS
func makeCounter() func() int {
    count := 0
    return func() int {
        count++
        return count
    }
}
counter := makeCounter()
counter() // 1
counter() // 2

// Passing functions as args (like .map(callback))
func apply(nums []int, fn func(int) int) []int {
    result := make([]int, len(nums))
    for i, n := range nums {
        result[i] = fn(n)
    }
    return result
}
```

**No exceptions, no `async/await`.** Functions that can fail return `(value, error)`. Functions that need to run concurrently are launched with `go` and communicate via channels — covered in section 12.

---

## 6. Arrays, Slices, Maps

### Arrays — fixed size (rarely used directly)

```go
var arr [5]int // fixed length 5, all zeros
arr[0] = 10
```

### Slices — the JS Array you actually use

```go
// Like a dynamic array
s := []int{1, 2, 3}
s = append(s, 4) // like .push() but returns a NEW slice (reassign!)

// make() with length/capacity, like new Array(len)
s2 := make([]int, 0, 10) // len=0, cap=10 (pre-allocated for performance)

// Slicing (same idea as Array.slice / Python slicing)
s3 := s[1:3] // elements at index 1,2

len(s)   // length
cap(s)   // capacity (underlying array size)

// IMPORTANT gotcha: slices share underlying arrays.
// Mutating a sub-slice can mutate the original — unlike JS's .slice() which copies.
```

Common slice patterns:

```go
// map() equivalent
doubled := make([]int, len(nums))
for i, n := range nums {
    doubled[i] = n * 2
}

// filter() equivalent
var evens []int
for _, n := range nums {
    if n%2 == 0 {
        evens = append(evens, n)
    }
}

// There's no built-in .map/.filter/.reduce — Go favors explicit loops.
// (Libraries like `samber/lo` add these if you want them, but idiomatic Go writes loops.)
```

### Maps — like a JS `Object` / `Map`

```go
m := map[string]int{"a": 1, "b": 2}
m["c"] = 3          // set
val := m["a"]        // get -> 1
val, ok := m["z"]     // ok = false if key doesn't exist (like Map.has())
delete(m, "a")        // delete key

// Zero value for a missing key is the type's zero value, NOT undefined
missing := m["doesNotExist"] // 0, not an error, not undefined
```

---

## 7. Structs & Methods

Go has **no classes**. You get `structs` (like a shape/interface in TS) + methods attached to them + composition instead of inheritance.

```go
type User struct {
    Name  string
    Email string
    Age   int
}

// Creating instances
u1 := User{Name: "Alice", Email: "alice@x.com", Age: 30}
u2 := User{"Bob", "bob@x.com", 25} // positional, less readable, avoid

// Methods — attached via a "receiver"
func (u User) Greet() string {
    return "Hi, I'm " + u.Name
}
fmt.Println(u1.Greet())

// Pointer receiver — use this when you need to MUTATE the struct
// (like passing `this` by reference; value receivers get a COPY)
func (u *User) Birthday() {
    u.Age++
}
u1.Birthday() // Go auto-takes the address, no need for (&u1).Birthday()

// Composition instead of inheritance (embedding)
type Admin struct {
    User        // embedded struct — "inherits" User's fields/methods
    Permissions []string
}

a := Admin{User: User{Name: "Root"}, Permissions: []string{"all"}}
a.Greet()      // works! promoted from User
a.Name         // also promoted, no need for a.User.Name
```

**Mongoose comparison:** where you'd define a Mongoose schema with methods, here you define a struct with attached functions. No inheritance hierarchies — Go developers explicitly favor "has-a" (embedding/composition) over "is-a" (classical inheritance).

### JSON tags (like Mongoose schema field mapping)

```go
type User struct {
    ID    string `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email,omitempty"` // omit if empty, like Mongoose's default behavior
    Age   int    `json:"-"`               // never serialize this field
}
```

---

## 8. Interfaces

This is Go's version of duck typing / TypeScript structural typing — and it's central to idiomatic Go.

```go
type Shape interface {
    Area() float64
}

type Circle struct{ Radius float64 }
func (c Circle) Area() float64 { return 3.14159 * c.Radius * c.Radius }

type Rectangle struct{ W, H float64 }
func (r Rectangle) Area() float64 { return r.W * r.H }

// Any type with an Area() method automatically satisfies Shape.
// No `implements` keyword needed — this is implicit, structural typing.
func printArea(s Shape) {
    fmt.Println(s.Area())
}

printArea(Circle{Radius: 2})
printArea(Rectangle{W: 3, H: 4})
```

The empty interface `interface{}` (or `any` in modern Go) is like TS's `any` / JS's untyped nature:

```go
func describe(i any) {
    fmt.Printf("value: %v, type: %T\n", i, i)
}
```

Type assertions (like a runtime type check):

```go
var i any = "hello"
s, ok := i.(string) // ok=true, s="hello"

// Type switch (like discriminated unions in TS)
switch v := i.(type) {
case string:
    fmt.Println("it's a string:", v)
case int:
    fmt.Println("it's an int:", v)
default:
    fmt.Println("unknown type")
}
```

**Interview tip:** "Accept interfaces, return structs" is a famous Go idiom — write functions that accept the smallest interface needed (flexible input), but return concrete types (predictable output).

---

## 9. Error Handling

No try/catch. Errors are values, returned explicitly, checked explicitly.

```go
import "errors"

func doSomething() error {
    return errors.New("something failed")
}

// The standard pattern — you'll type this a thousand times
result, err := doSomething()
if err != nil {
    // handle it, log it, or bubble it up
    return err
}

// Wrapping errors with context (like adding a stack trace / extra message)
func readConfig() error {
    err := doSomething()
    if err != nil {
        return fmt.Errorf("readConfig failed: %w", err) // %w wraps the original error
    }
    return nil
}

// Unwrapping / checking specific errors
if errors.Is(err, sql.ErrNoRows) { ... }

var myErr *MyCustomError
if errors.As(err, &myErr) { ... }
```

Custom errors (like custom Error classes in JS):

```go
type ValidationError struct {
    Field string
    Msg   string
}
func (e *ValidationError) Error() string {
    return fmt.Sprintf("%s: %s", e.Field, e.Msg)
}
```

`panic`/`recover` exist (Go's closest thing to throw/catch) but are reserved for truly unrecoverable situations (like a corrupted internal state), never for normal control flow like a failed HTTP request or a missing DB row.

```go
func safeDivide(a, b int) (result int, err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("recovered: %v", r)
        }
    }()
    return a / b, nil // panics if b == 0
}
```

`defer` itself is huge in Go — like a guaranteed `finally` block, runs when the function returns, used constantly for cleanup:

```go
func readFile() {
    f, _ := os.Open("file.txt")
    defer f.Close() // guaranteed to run, even if the function panics or returns early
    // ... use f
}
```

---

## 10. Pointers

You've avoided these in JS because JS handles references for you. Go makes it explicit.

```go
x := 10
p := &x        // p is a pointer to x, holds x's memory address
fmt.Println(*p) // dereference: prints 10
*p = 20         // mutate the original x through the pointer
fmt.Println(x)  // 20

// Why it matters: value vs reference semantics
func increment(n int) { n++ }         // gets a COPY, original unchanged
func incrementPtr(n *int) { *n++ }    // gets the ADDRESS, original changes

a := 5
increment(a)     // a is still 5
incrementPtr(&a) // a is now 6
```

Rule of thumb: structs are passed by value by default (copied). If a function needs to mutate the caller's data, or the struct is large (avoid copying overhead), use a pointer receiver/parameter — this is the direct equivalent of "pass by reference" you get automatically with JS objects.

---

## 11. Packages & Modules

```go
// go.mod (like package.json)
module github.com/yourname/myapp

go 1.22

require (
    github.com/gin-gonic/gin v1.9.1
)
```

```bash
go get github.com/gin-gonic/gin   # like npm install
go mod tidy                        # like npm prune + install, cleans up go.mod/go.sum
```

File/folder = package convention:

```
myapp/
  main.go          package main
  handlers/
    user.go        package handlers
  models/
    user.go        package models
```

```go
// models/user.go
package models

type User struct { Name string }

// main.go
package main
import "github.com/yourname/myapp/models"

func main() {
    u := models.User{Name: "Alice"}
}
```

Capitalization = visibility (this trips up every JS dev at first):
- `func Foo()` — Exported (public), like `export` in JS
- `func foo()` — unexported (private to the package), like not exporting it

There's no explicit `export` keyword — the capital letter IS the export.

---

## 12. Goroutines & Channels

This is Go's headline feature and the #1 thing interviewers probe for.

### Goroutines — lightweight threads

```go
func sayHello() {
    fmt.Println("hello")
}

func main() {
    go sayHello() // runs concurrently, like not awaiting a promise
    time.Sleep(time.Second) // without this, main() might exit before sayHello runs
}
```

Compare to Node: `setTimeout`/Promises give you concurrency on ONE thread via the event loop. Goroutines are scheduled across OS threads/CPU cores by the Go runtime — this is real parallelism, not just interleaving.

### Channels — typed pipes for goroutines to talk to each other

```go
ch := make(chan int) // unbuffered channel

go func() {
    ch <- 42 // send
}()

value := <-ch // receive (blocks until a value arrives)
fmt.Println(value)

// Buffered channel (like a queue with capacity)
ch2 := make(chan string, 3)
ch2 <- "a"
ch2 <- "b"
// doesn't block until buffer is full

// Closing a channel (signals "no more values")
close(ch2)

// Range over a channel (consume until closed — like reading a stream until 'end')
for msg := range ch2 {
    fmt.Println(msg)
}
```

### The classic worker pool pattern (VERY common interview question)

```go
func worker(id int, jobs <-chan int, results chan<- int) {
    for j := range jobs {
        results <- j * 2
    }
}

func main() {
    jobs := make(chan int, 100)
    results := make(chan int, 100)

    // spin up 3 workers
    for w := 1; w <= 3; w++ {
        go worker(w, jobs, results)
    }

    for j := 1; j <= 5; j++ {
        jobs <- j
    }
    close(jobs)

    for a := 1; a <= 5; a++ {
        fmt.Println(<-results)
    }
}
```

### select — like Promise.race for channels

```go
select {
case msg1 := <-ch1:
    fmt.Println("ch1:", msg1)
case msg2 := <-ch2:
    fmt.Println("ch2:", msg2)
case <-time.After(1 * time.Second):
    fmt.Println("timeout")
default:
    fmt.Println("no channel ready") // non-blocking
}
```

**Go's philosophy (interview-quotable):** "Do not communicate by sharing memory; instead, share memory by communicating." Channels pass ownership of data between goroutines rather than having multiple goroutines lock/read/write the same variable — this avoids most classic race conditions by design.

---

## 13. sync Package

For cases where channels are overkill and you just need to protect shared state (like a global cache/counter).

```go
import "sync"

// WaitGroup — like Promise.all, wait for N goroutines to finish
var wg sync.WaitGroup
for i := 0; i < 5; i++ {
    wg.Add(1)
    go func(n int) {
        defer wg.Done()
        fmt.Println(n)
    }(i)
}
wg.Wait() // blocks until all 5 are done

// Mutex — locking shared state (like a critical section)
var mu sync.Mutex
counter := 0
increment := func() {
    mu.Lock()
    defer mu.Unlock()
    counter++
}

// Run `go run -race main.go` to detect race conditions — Go's built-in race detector,
// something JS simply doesn't need but Go absolutely does.
```

---

## 14. Context

Used everywhere in real Go web servers — think of it as a bundled `AbortController` + request-scoped values, passed down the call chain.

```go
import "context"

func doWork(ctx context.Context) error {
    select {
    case <-time.After(5 * time.Second):
        return nil
    case <-ctx.Done():
        return ctx.Err() // caller cancelled or timed out
    }
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel()
    doWork(ctx) // will return ctx.Err() after 2s, like AbortSignal.timeout(2000)
}
```

In an HTTP server, `r.Context()` gives you a context tied to the request lifecycle — if the client disconnects, everything downstream (DB queries, API calls) can bail out early, same idea as cancelling a `fetch` with `AbortController`.

---

## 15. Generics

Added in Go 1.18 — like TypeScript generics.

```go
func Map[T, U any](items []T, fn func(T) U) []U {
    result := make([]U, len(items))
    for i, v := range items {
        result[i] = fn(v)
    }
    return result
}

doubled := Map([]int{1, 2, 3}, func(n int) int { return n * 2 })
strs := Map([]int{1, 2, 3}, func(n int) string { return fmt.Sprint(n) })

// Generic types
type Stack[T any] struct {
    items []T
}
func (s *Stack[T]) Push(item T) { s.items = append(s.items, item) }
func (s *Stack[T]) Pop() (T, bool) {
    var zero T
    if len(s.items) == 0 {
        return zero, false
    }
    item := s.items[len(s.items)-1]
    s.items = s.items[:len(s.items)-1]
    return item, true
}

// Constraints (like `T extends number`)
type Number interface {
    int | int64 | float64
}
func Sum[T Number](nums []T) T {
    var total T
    for _, n := range nums {
        total += n
    }
    return total
}
```

---

## 16. Testing

Testing is built into the standard library — no Jest/Mocha needed.

```go
// math.go
package math

func Add(a, b int) int { return a + b }
```

```go
// math_test.go — file MUST end in _test.go
package math

import "testing"

func TestAdd(t *testing.T) {
    result := Add(2, 3)
    expected := 5
    if result != expected {
        t.Errorf("Add(2,3) = %d; want %d", result, expected)
    }
}

// Table-driven tests — the idiomatic Go pattern (like Jest's test.each)
func TestAddTable(t *testing.T) {
    cases := []struct {
        a, b, want int
    }{
        {1, 1, 2},
        {2, 3, 5},
        {-1, 1, 0},
    }
    for _, c := range cases {
        got := Add(c.a, c.b)
        if got != c.want {
            t.Errorf("Add(%d,%d) = %d; want %d", c.a, c.b, got, c.want)
        }
    }
}
```

```bash
go test ./...            # run all tests, like `npm test`
go test -v ./...         # verbose
go test -cover ./...     # coverage report
go test -race ./...      # race detector
```

For HTTP handler testing, use `net/http/httptest` (equivalent to `supertest` in Node):

```go
func TestHandler(t *testing.T) {
    req := httptest.NewRequest("GET", "/users", nil)
    w := httptest.NewRecorder()
    userHandler(w, req)

    if w.Code != http.StatusOK {
        t.Errorf("expected 200, got %d", w.Code)
    }
}
```

---

## 17. Building a REST API

### Level 1: Pure standard library (no Express needed — net/http is already solid)

```go
package main

import (
    "encoding/json"
    "net/http"
)

type User struct {
    ID   string `json:"id"`
    Name string `json:"name"`
}

func getUsers(w http.ResponseWriter, r *http.Request) {
    users := []User{{ID: "1", Name: "Alice"}}
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(users)
}

func createUser(w http.ResponseWriter, r *http.Request) {
    var u User
    if err := json.NewDecoder(r.Body).Decode(&u); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(u)
}

func main() {
    mux := http.NewServeMux() // like express.Router()
    mux.HandleFunc("GET /users", getUsers)   // Go 1.22+ supports method-based routing
    mux.HandleFunc("POST /users", createUser)

    http.ListenAndServe(":8080", mux) // like app.listen(8080)
}
```

### Level 2: Middleware (like Express middleware)

```go
func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        next.ServeHTTP(w, r) // like calling next()
        log.Println(r.Method, r.URL.Path, time.Since(start))
    })
}

mux.Handle("/users", loggingMiddleware(http.HandlerFunc(getUsers)))
```

### Level 3: Popular frameworks (like choosing Express/Fastify/Koa)

- **Gin** — most popular, Express-like ergonomics
- **Chi** — lightweight router, close to stdlib, composable middleware
- **Fiber** — Express-inspired API, built on fasthttp

```go
// Gin example
import "github.com/gin-gonic/gin"

func main() {
    r := gin.Default()
    r.GET("/users/:id", func(c *gin.Context) {
        id := c.Param("id")
        c.JSON(200, gin.H{"id": id})
    })
    r.Run(":8080")
}
```

Request validation (like Zod/Joi):

```go
type CreateUserRequest struct {
    Name  string `json:"name" binding:"required"`
    Email string `json:"email" binding:"required,email"`
}

func createUser(c *gin.Context) {
    var req CreateUserRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(400, gin.H{"error": err.Error()})
        return
    }
    // ...
}
```

---

## 18. Databases

### database/sql (stdlib, like the `pg`/`mysql2` driver in Node — low level)

```go
import (
    "database/sql"
    _ "github.com/lib/pq" // postgres driver, imported for side effects
)

db, err := sql.Open("postgres", "connection_string")
defer db.Close()

// Query (like knex/raw SQL)
rows, err := db.Query("SELECT id, name FROM users WHERE age > $1", 18)
defer rows.Close()
for rows.Next() {
    var id, name string
    rows.Scan(&id, &name)
}

// Exec for INSERT/UPDATE/DELETE
_, err = db.Exec("INSERT INTO users (name) VALUES ($1)", "Alice")
```

### GORM (the Mongoose/Prisma/Sequelize of Go)

```go
import "gorm.io/gorm"

type User struct {
    gorm.Model         // adds ID, CreatedAt, UpdatedAt, DeletedAt automatically
    Name  string
    Email string `gorm:"unique"`
}

db.AutoMigrate(&User{})           // like Mongoose schema sync / Prisma migrate

db.Create(&User{Name: "Alice"})    // like User.create()
var users []User
db.Where("name = ?", "Alice").Find(&users) // like User.find({name: "Alice"})
db.First(&user, 1)                  // like User.findById(1)
db.Save(&user)                      // like user.save()
db.Delete(&user, 1)                 // like User.findByIdAndDelete(1)
```

### MongoDB (using the official Go driver — for direct MERN parity)

```go
import "go.mongodb.org/mongo-driver/mongo"

client, _ := mongo.Connect(ctx, options.Client().ApplyURI("mongodb://localhost:27017"))
collection := client.Database("mydb").Collection("users")

// Insert (like Model.create())
collection.InsertOne(ctx, User{Name: "Alice"})

// Find (like Model.find())
cursor, _ := collection.Find(ctx, bson.M{"name": "Alice"})
var results []User
cursor.All(ctx, &results)
```

---

## 19. Project Structure

A common, idiomatic layout (loosely modeled after community standards):

```
myapp/
├── go.mod
├── cmd/
│   └── api/
│       └── main.go          # entrypoint, like index.js / server.js
├── internal/                 # private code, can't be imported by other modules
│   ├── handlers/              # like Express route handlers/controllers
│   │   └── user_handler.go
│   ├── models/                # like Mongoose models
│   │   └── user.go
│   ├── repository/            # like your DB access layer / DAO
│   │   └── user_repo.go
│   ├── service/                # business logic layer
│   │   └── user_service.go
│   └── middleware/
│       └── auth.go
├── pkg/                       # code that's OK to be imported externally
├── config/
│   └── config.go
└── .env
```

Layered architecture maps directly to what you already do in Express:
`handler (controller) → service (business logic) → repository (DB access)`

---

## 20. Interview Cheat Sheet

**Concurrency**
- Explain goroutines vs OS threads: goroutines are cheap (~2KB stack, grows dynamically), multiplexed onto OS threads by the Go scheduler (M:N scheduling).
- Explain unbuffered vs buffered channels: unbuffered blocks sender until a receiver is ready (synchronous handoff); buffered allows sending up to capacity without blocking.
- Know the worker pool pattern (section 12) cold — it's the most common live-coding ask.
- Know `sync.WaitGroup` vs channels for coordination, and when to use each.
- Be able to explain a data race and how `go run -race` catches it.

**Language design**
- Why no exceptions? Errors as values force explicit handling at every call site; nothing fails silently.
- Why no inheritance? Composition (embedding) + interfaces give flexibility without deep, fragile hierarchies.
- Explain the empty interface `interface{}`/`any` and its tradeoffs (loses type safety, like `any` in TS).
- Explain value vs pointer receivers and when each is appropriate.
- Explain slices: they're a header (pointer, length, capacity) over an underlying array — this explains the "slices share memory" gotcha.

**Practical**
- Explain how you'd structure a Go REST API (handler → service → repository).
- Explain `context.Context` and its role in cancellation/timeouts across a call chain.
- Explain how Go compiles to a static binary and why that simplifies deployment vs a Node app needing a runtime.
- Know `defer`, and that deferred calls run in LIFO order.
- Be ready to write a basic goroutine + channel example live — this comes up constantly.

**Common "gotcha" questions**
- What happens if you range over a slice and mutate it inside the loop? (Behavior can be surprising — ranging captures values, not live references in older Go; Go 1.22+ changed loop variable semantics per-iteration.)
- Why does appending to a slice sometimes not affect the original? (Depends on whether capacity was exceeded, triggering a new underlying array.)
- What's the zero value of a slice/map/pointer, and what happens if you read/write to a nil map vs a nil slice? (Reading a nil map is safe and returns zero value; **writing to a nil map panics**. Reading a nil slice is safe; appending to a nil slice works fine, unlike maps.)

---

