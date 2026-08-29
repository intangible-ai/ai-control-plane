# Slice v1-00b — Go for a Node developer

**No project code. ~2 hours. Read it, then type the five programs at the end.**

You said Go looks scary and you are at zero. Both are fine. This document exists to replace a
vague fear with a specific, finite list — because "I don't know Go" is frightening and
"I don't know 9 things, here they are" is a Tuesday.

---

## 1. The claim, stated plainly: Go is a small language

This is not encouragement, it is a measurable fact.

| | JavaScript | Go |
|---|---|---|
| Keywords | ~64 reserved words | **25** |
| Ways to declare a variable | `var`, `let`, `const` | `var`, `:=` |
| Inheritance model | prototype chain + `class` sugar | **none** |
| `this` binding rules | 5 of them, plus arrow-function exceptions | **no `this`** |
| Async model | callbacks → Promises → async/await | **none — just write blocking code** |
| Equality | `==` vs `===` vs `Object.is` | **`==`** |
| Truthiness | `0`, `""`, `NaN`, `null`, `undefined`, `[]`... | **only `bool` is allowed in `if`** |
| Null-ish | `null` **and** `undefined` | **`nil`** |

Go was designed at Google with an explicit goal: a language a new hire could be productive in
within weeks, and that a large team could read without surprises. Every "missing" feature above is
missing **on purpose**.

**The things that make JavaScript hard do not exist in Go.** You are not adding a hard language on
top of an easy one. You are trading a large, quirky language for a small, blunt one.

What you actually have to learn is a short list. Here it is, complete.

---

## 2. The nine differences that matter

### 2.1 Types are written down

```js
// JS
function add(a, b) { return a + b; }
```
```go
// Go — the types are the only new part
func add(a int, b int) int { return a + b }
```

If you have used TypeScript, you already have this. If not: the type goes **after** the name, and
the return type goes after the parameters. That is the entire syntax rule.

Inside a function, `:=` infers the type so you rarely write it:

```go
count := 0            // int
name := "gateway"     // string
ok := true            // bool
```

`:=` declares **and** assigns. `=` only assigns to something already declared. That is the one
gotcha, and the compiler tells you immediately.

### 2.2 A struct is a TypeScript interface that exists at runtime

```ts
// TS — erased at compile time
interface ChatRequest { model: string; prompt: string; }
```
```go
// Go — a real thing in memory
type ChatRequest struct {
	Model  string
	Prompt string
}

req := ChatRequest{Model: "claude-haiku-4-5", Prompt: "hello"}
fmt.Println(req.Model)
```

**Capital letter = exported (public). Lowercase = package-private.** That is Go's entire
visibility system. No `public`/`private` keywords. `Model` is visible outside the package;
`model` would not be.

### 2.3 Struct tags: the backtick thing that looks like noise

```go
type ChatRequest struct {
	Model  string `json:"model"`
	Prompt string `json:"prompt"`
}
```

In Node, `JSON.parse` gives you whatever keys were in the JSON. Go needs to know which JSON key
maps to which field, because the field is `Model` (capital, exported) but the JSON says `"model"`.
The backtick string is that mapping. Copy the pattern; there is nothing deeper to understand.

### 2.4 Errors are values, not exceptions

This is the one people complain about, and it is the one worth defending.

```js
// Node — the error path is invisible until it isn't
const data = await fetchThing();   // might throw. Does it? Read the docs. Or don't.
```
```go
// Go — the error path is in your face, at the exact line that can fail
data, err := fetchThing()
if err != nil {
	return err          // or log it, or wrap it, or retry — your choice, made here
}
```

A Go function returns **two** values: the result and an error. `nil` error means success.

Yes, you write `if err != nil` a lot. Consider what it replaces: a `try/catch` wrapped around
fifteen lines where any of them might have thrown, and you cannot tell which. Go makes you decide
what to do about each failure *at the point it can happen*.

**For this project specifically that is not a style preference — it is the subject matter.** The
whole portfolio thesis is "correctness under partial failure." A language that makes every failure
point visible is the correct tool for a project about failure points.

Two shortcuts you will use constantly:

```go
// wrap an error with context as it travels up
if err != nil {
	return fmt.Errorf("calling provider: %w", err)
}

// in main() only, when there is nothing sensible to do but die
log.Fatal(err)
```

### 2.5 Pointers — you need about 20% of them

`&x` means "the address of x". `*T` in a type means "a pointer to a T".

You need pointers in exactly three situations in this project:

1. **When a function must fill something in for you:**
   ```go
   var req ChatRequest
   json.NewDecoder(r.Body).Decode(&req)   // &req so Decode can write into it
   ```
   Mentally: in JS you pass an object and the callee mutates it. Go makes you say so.

2. **On methods that modify the receiver:**
   ```go
   func (s *Server) Start() error { ... }   // *Server, because Start changes s
   ```

3. **To avoid copying a big struct.** Rare at our size. Ignore it for now.

You will **not** need pointer arithmetic, `malloc`, `free`, or manual memory management. Go has a
garbage collector, exactly like Node. Pointers here are about *who can modify what*, not about
memory safety.

### 2.6 No async/await — and this is a simplification

```js
// Node: everything I/O touches becomes async, and it spreads
async function handle(req) {
  const docs = await retrieve(req.q);      // function must be async
  const reply = await callModel(docs);     // caller must be async
  return reply;                            // and its caller. Forever.
}
```
```go
// Go: just write it
func handle(q string) (string, error) {
	docs, err := retrieve(q)
	if err != nil { return "", err }
	return callModel(docs)
}
```

Go's runtime gives every incoming HTTP request its own **goroutine** — a very cheap thread (a few
KB, not a few MB). When your code blocks on I/O, the runtime parks that goroutine and runs
another. You get concurrency without ever writing concurrent-looking code.

Node solved the same problem by making *you* write the non-blocking style. Go solved it in the
runtime. **You are deleting a concept, not adding one.**

When you *do* want two things at once, it is one word:

```go
go doSomething()    // runs concurrently. That's it.
```

We will use that in the load generator, and I will teach it there.

### 2.7 Interfaces are implicit — and this is why the provider abstraction works

```go
// declare what you need
type Provider interface {
	Chat(ctx context.Context, req ChatRequest) (ChatResponse, error)
}

// any type with that method IS a Provider. No "implements" keyword.
type MockProvider struct{}
func (m MockProvider) Chat(ctx context.Context, req ChatRequest) (ChatResponse, error) { ... }

type AnthropicProvider struct{ apiKey string }
func (a AnthropicProvider) Chat(ctx context.Context, req ChatRequest) (ChatResponse, error) { ... }
```

Nothing declares that it implements `Provider`. If the method signature matches, it satisfies the
interface. This is duck typing, checked at compile time.

Why it matters here: **the interface is defined by the consumer, not the producer.** Our gateway
says "I need something with a `Chat` method." Adapters satisfy it without knowing our interface
exists. That is what makes "provider is a runtime config value, never an import" structurally
true rather than aspirational.

### 2.8 `context.Context` — the parameter you will see everywhere

```go
func (s *Server) handleChat(w http.ResponseWriter, r *http.Request) {
	resp, err := s.provider.Chat(r.Context(), req)
	//                          ^^^^^^^^^^^ just pass it along
}
```

`context` carries **cancellation and deadlines** down a call chain. When an HTTP client
disconnects, `r.Context()` is cancelled, and everything downstream that respects it stops.

For us this is not academic: **if a user closes the tab mid-stream, we want to stop paying the
provider for tokens nobody will read.** Context is how that signal travels.

For now: it is the first parameter of most functions, and you pass the one you were given. That
is 90% of what you need.

### 2.9 The toolchain is smaller than npm

| npm | Go |
|---|---|
| `npm init` | `go mod init github.com/intangible-ai/ai-control-plane` |
| `npm install x` | `go get x` |
| `node index.js` | `go run ./cmd/gateway` |
| build step, bundler | `go build ./...` → **one static binary, no runtime needed** |
| prettier + eslint config debates | `gofmt` — built in, no options, no arguments |
| `package.json` + `package-lock.json` | `go.mod` + `go.sum` |
| `node_modules/` (400 MB) | a shared module cache |

`gofmt` deserves a note: Go has **one** formatting style, enforced by a tool in the standard
distribution. There is no debate, no config file, no team argument. VS Code's Go extension runs it
on save. You will never think about formatting again.

---

## 3. What you can safely ignore, possibly forever

Fear grows in proportion to how much you think you must learn. Here is the list you can put down:

- **Generics** — added in 1.18, genuinely useful, and we will not need them in v1.
- **Channels and `select`** — the famous Go concurrency thing. We use exactly one, in slice
  `v1-07`, as a bounded queue, and I will teach it there. You do not need it before that.
- **`sync.Mutex`, `WaitGroup`, atomics** — a taste in the load generator, taught in place.
- **Reflection** (`reflect`) — no.
- **Struct embedding, method sets, pointer-vs-value receiver subtleties** — real Go trivia. The
  compiler will tell you when you get it wrong, and the fix is always mechanical.
- **`unsafe`, cgo, build tags, assembly** — no.
- **Any web framework** — Gin, Echo, Fiber, Chi. Not needed. Not now, possibly not ever.

**Everything you need for slice `v1-01` is in section 2 above.** That is not a simplification to
make you feel better; go back and look at the 35-line handler — every construct in it is covered.

---

## 4. Your 2-hour on-ramp

Do these in order, in a scratch folder, **not** in the project. Type them by hand. Do not paste —
the muscle memory is the point, and the compiler errors you cause by typos are the fastest way to
learn what the compiler is telling you.

Setup, once, in PowerShell:

```powershell
mkdir D:\go-scratch
```

```powershell
cd D:\go-scratch
```

```powershell
go mod init scratch
```

### Program 1 — it runs (10 min)

`D:\go-scratch\main.go`

```go
package main

import "fmt"

func main() {
	fmt.Println("hello from go")
}
```

```powershell
go run .
```

### Program 2 — types, structs, and the visibility rule (20 min)

```go
package main

import "fmt"

type Token struct {
	Kind  string
	Count int
}

func total(tokens []Token) int {
	sum := 0
	for _, t := range tokens {
		sum += t.Count
	}
	return sum
}

func main() {
	usage := []Token{
		{Kind: "input", Count: 1024},
		{Kind: "output", Count: 312},
		{Kind: "cache_read", Count: 8192},
	}
	for _, t := range usage {
		fmt.Printf("%-12s %6d\n", t.Kind, t.Count)
	}
	fmt.Println("total:", total(usage))
}
```

New things to notice: `[]Token` is a slice (a JS array). `range` is `for...of`. `_` means "I am
deliberately ignoring this value" — the index, here — and Go **requires** you to say so, because
unused variables are a compile error. That strictness feels hostile for a day and then you realise
it has been quietly deleting your dead code.

### Program 3 — errors as values (20 min)

```go
package main

import (
	"errors"
	"fmt"
)

var pricePerMillion = map[string]float64{
	"claude-haiku-4-5": 1.00,
	"claude-sonnet-5":  3.00,
}

func inputCost(model string, tokens int) (float64, error) {
	price, ok := pricePerMillion[model]
	if !ok {
		return 0, fmt.Errorf("unknown model %q", model)
	}
	if tokens < 0 {
		return 0, errors.New("negative tokens")
	}
	return float64(tokens) / 1_000_000 * price, nil
}

func main() {
	for _, m := range []string{"claude-haiku-4-5", "gpt-9"} {
		cost, err := inputCost(m, 1_200_000)
		if err != nil {
			fmt.Println("error:", err)
			continue
		}
		fmt.Printf("%s: $%.4f\n", m, cost)
	}
}
```

The `value, ok := someMap[key]` pattern is Go's answer to `undefined` — a second boolean telling
you whether the key existed. No `undefined` sneaking through three call frames.

### Program 4 — interfaces, and the provider abstraction in miniature (30 min)

```go
package main

import "fmt"

type Provider interface {
	Chat(prompt string) (string, error)
}

type MockProvider struct {
	Reply string
}

func (m MockProvider) Chat(prompt string) (string, error) {
	return m.Reply, nil
}

type EchoProvider struct{}

func (e EchoProvider) Chat(prompt string) (string, error) {
	return "you said: " + prompt, nil
}

func ask(p Provider, prompt string) {
	reply, err := p.Chat(prompt)
	if err != nil {
		fmt.Println("error:", err)
		return
	}
	fmt.Println(reply)
}

func main() {
	providers := []Provider{
		MockProvider{Reply: "canned answer"},
		EchoProvider{},
	}
	for _, p := range providers {
		ask(p, "hello")
	}
}
```

**Look at what just happened.** `ask` has no idea which provider it got. Neither `MockProvider` nor
`EchoProvider` mentions the `Provider` interface anywhere. That is the entire provider abstraction
from the plan, in 35 lines. You have now written it.

### Program 5 — an HTTP server (30 min)

```go
package main

import (
	"encoding/json"
	"log"
	"net/http"
)

type ChatRequest struct {
	Model  string `json:"model"`
	Prompt string `json:"prompt"`
}

type ChatResponse struct {
	Reply string `json:"reply"`
	Model string `json:"model"`
}

func handleChat(w http.ResponseWriter, r *http.Request) {
	var req ChatRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		http.Error(w, "bad json", http.StatusBadRequest)
		return
	}
	if req.Prompt == "" {
		http.Error(w, "prompt required", http.StatusBadRequest)
		return
	}

	resp := ChatResponse{Reply: "you said: " + req.Prompt, Model: req.Model}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(resp)
}

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("POST /v1/chat", handleChat)
	log.Println("listening on :8080")
	log.Fatal(http.ListenAndServe(":8080", mux))
}
```

```powershell
go run .
```

In a second terminal:

```powershell
curl.exe -s -X POST localhost:8080/v1/chat -H "Content-Type: application/json" -d '{\"model\":\"claude-haiku-4-5\",\"prompt\":\"hello\"}'
```

*(Use `curl.exe`, not `curl` — in PowerShell, bare `curl` is an alias for `Invoke-WebRequest`,
which takes completely different arguments. This trips up everyone once.)*

**When that returns JSON, you have written 80% of slice `v1-01`.** Not an analogue of it. It.

---

## 5. Setup

```powershell
go version
```

Already confirmed present: **go1.25.5 windows/amd64**.

VS Code extension — install this before program 1:

```powershell
code --install-extension golang.go
```

On first opening a `.go` file it will offer to install `gopls` (the language server), `dlv`
(debugger) and friends. **Say yes.** After that you get autocomplete, jump-to-definition, inline
errors as you type, and format-on-save. The editor experience is genuinely better than the Node
one, because there is a single official language server rather than five competing ones.

---

## 6. The escape hatch, stated honestly

I am recommending Go for the gateway because CLAUDE.md §7 names exactly this shape —
"concurrency-heavy, latency-sensitive, long-running services (gateways, schedulers, proxies, load
generators)" — and because for a remote US AI-infrastructure role, Go on the resume separates you
from every other applicant who has Node. That is the actual hiring argument, and it is why the
friction is worth it.

But a fear with no exit is paralysing, so here is a real one.

**Decision point: after slice `v1-02`.** By then you will have written a server, an interface, two
adapters, and span emission — roughly 300 lines. That is enough to know whether Go is friction that
is fading or friction that is not.

If it is not fading, we switch the **gateway** to TypeScript and keep Go only for **`mockllm` and
`loadgen`** — about 250 lines total, mostly mechanical, and the two components where a
single-threaded event loop would genuinely become the bottleneck and quietly invalidate every
measurement. CLAUDE.md §7 explicitly blesses this: *"A polyglot project is fine and often correct —
a Go load generator against a Python service is a normal production shape, not a mess."*

That fallback costs us some of the hiring signal and none of the architecture. It is a real option,
not a consolation prize.

**What we do not do** is decide now, from fear, before you have typed any Go. Type the five
programs. Then decide with evidence — which is the same standard this entire portfolio holds
everything else to.

---

## Done means

- [ ] All five programs run on your machine
- [ ] You can say why `if err != nil` replaces `try/catch`, and what it buys in *this* project
- [ ] You can explain why `ask()` in program 4 does not know which provider it received
- [ ] You know which of the nine differences you still find uncomfortable — **name it, and I teach
      it again differently**

That last box is the important one. "I don't get pointers yet" is a solvable, twenty-minute
problem. "Go is scary" is not, because it has no edges. The point of this document is to give the
fear edges.

---

# Next

`v1-01` — walking skeleton. Program 5 above, plus a `Provider` interface, plus one span printed to
the terminal. First commit.
