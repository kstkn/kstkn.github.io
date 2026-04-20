---
layout: post
title: "Interfaces Belong to the Consumer: A Defense of Go's Most Misunderstood Convention"
date: 2026-04-20
---

> Go interfaces generally belong in the package that uses values of the interface type, not the package that implements those values. The implementing package should return concrete (usually pointer or struct) types: that way, new methods can be added to implementations without requiring extensive refactoring.
> — [Go Code Review Comments](https://go.dev/wiki/CodeReviewComments#interfaces)

Developers arriving in Go from Java, C#, or PHP tend to produce a familiar code: a package `storage` that exports an interface `Storage` alongside a struct `PostgresStorage` that implements it. Any consumer then imports `storage.Storage` and receives a `storage.PostgresStorage` from a constructor. It feels clean and like good object-oriented design. It is almost always wrong in Go.

The convention above — often summarized as **"accept interfaces, return structs"** - is not a matter of taste. It falls out of Go's type system, its dependency model, and its approach to software evolution. Here is why it holds up.

## 1. Interfaces in Go are satisfied structurally, not nominally

In Java or C#, a class must explicitly declare `implements Storage`. The interface and the implementation are linked by name, and the compiler enforces that link. A type cannot satisfy an interface it has never heard of.

Go inverts this. A type satisfies an interface by having the right method set. There is no `implements` keyword, no declaration of intent. A `*os.File` satisfies `io.Reader` without `os` knowing `io.Reader` exists at that call site, and without `io` knowing anything about files.

This single language decision changes where interfaces belong. Because the implementing type doesn't need to import the interface to satisfy it, **the interface has no reason to live near the implementation**. It is free to reside wherever it is most useful - which is the package that needs to describe its dependencies.

If you put the interface in the implementing package anyway, you're paying the coordination cost of nominal typing while using a language that gave you structural typing.

## 2. Interfaces describe what the consumer needs, not what the producer provides

Consider we are implementing a persistence for users. We define a `UserStorage` and declare its functionality at the producer side:

```go
package storage

type User struct{ /* ... */ }

type UserStorage interface {
    GetByID(id int) (User, error)
    GetByName(name string) (User, error)
    Add(u User) error
    Remove(id int) error
    Find(f func(u User) bool) ([]User, error)
}

type MemoryUserStorage struct {
    store []User
}

func (s *MemoryUserStorage) GetByID(id int) (User, error)          { /* ... */ }
func (s *MemoryUserStorage) GetByName(n string) (User, error)      { /* ... */ }
func (s *MemoryUserStorage) Add(u User) error                      { /* ... */ }
func (s *MemoryUserStorage) Remove(id int) error                   { /* ... */ }
func (s *MemoryUserStorage) Find(f func(User) bool) ([]User, error) { /* ... */ }
```

Now let's say there's a consumer in another package that needs to print a user's info:

```go
package report

import "storage"

func PrintUser(s storage.UserStorage, id int) error {
    u, err := s.GetByID(id)
    if err != nil {
        return err
    }
    fmt.Println(u)
    return nil
}
```

`PrintUser` uses exactly one of `UserStorage`'s five methods. The other four — `GetByName`, `Add`, `Remove`, `Find` — are dead weight at this call site. But because the interface was authored by the producer, `PrintUser` has to accept the whole contract. A unit test has to stub four methods it will never call. A reviewer reading the signature cannot tell which operations `PrintUser` actually performs without opening the body. And `report` now imports `storage`, which in a real codebase is likely to drag in a database driver, a connection pool, and migration code that `PrintUser` has no business knowing about.

Turn it around and the contract becomes honest:

```go
package report

type userReader interface {
    GetByID(id int) (storage.User, error)
}

func PrintUser(s userReader, id int) error { /* ... */ }
```

One method. The signature now tells the reader exactly what `PrintUser` touches. `*MemoryUserStorage` still satisfies it — structural typing does not care that `userGetter` was declared in a different package from the implementation. Tests stub a single method. And if `storage` later adds `BulkUpdate`, `StreamAll`, or a tenant-scoped `GetByIDForTenant`, `PrintUser` doesn't notice.

Now picture the rest of the codebase. A `welcome` package sends onboarding email and needs `GetByID`. A `search` package lists people by name-prefix and needs `Find`. An `admin` tool bans accounts and needs `GetByName` and `Remove`. A `signup` handler needs `Add`. Four consumers, four disjoint contracts, each one or two methods wide. None of them is `UserStorage`. If the producer owns the interface, every caller either lies about its dependencies by accepting all five methods, or invents its own narrower interface anyway — which is the convention, just reached the slow way.

**The consumer knows what it needs. The producer does not.** The `storage` package, at the moment it declares `UserStorage`, cannot know which callers will print, search, onboard, or clean up — and cannot know which callers will exist next quarter. Interfaces written by the producer inevitably either over-specify (forcing callers to depend on methods they don't use) or under-specify (missing what a particular caller needs). Interfaces written by the consumer fit the call site exactly, because they are authored against real usage. This anti-pattern — producer packages declaring interfaces ahead of any concrete consumer need — is commonly called [*interface pollution*](https://100go.co/5-interface-pollution/).

This is why `io.Reader` has one method, not twelve. It wasn't designed by whoever wrote `os.File`. It was designed by someone who needed to read bytes — and the same discipline applies to a `userGetter` that only needs to look one user up by ID.

## 3. Returning concrete types preserves the right to evolve

Here is the core refactoring argument from the convention. Suppose `database` exports:

```go
package database

type DB interface {
    Query(string) (Rows, error)
    Exec(string) (Result, error)
}

func Open(dsn string) DB { ... }
```

Six months later, you want to add `BeginTx`. You have two choices, both bad:

1. **Add it to the interface.** Every type that satisfies `DB` — including test doubles, third-party wrappers, and in-memory fakes across every repository — now fails to compile until its author adds a `BeginTx` method. You have coupled the evolution of your package to the evolution of everyone else's code.
2. **Don't add it.** You now have an implementation with a method no one can call without a type assertion, which defeats the interface and confuses readers.

Now compare the concrete-return version:

```go
package database

type DB struct { ... }

func (d *DB) Query(string) (Rows, error) { ... }
func (d *DB) Exec(string)  (Result, error) { ... }

func Open(dsn string) *DB { ... }
```

Adding `BeginTx` is a one-line, non-breaking change. Existing callers who only used `Query` and `Exec` notice nothing. New callers who need `BeginTx` use it. Consumers who want to mock `*DB` in tests define their own minimal interface — `type querier interface { Query(string) (Rows, error) }` — that captures exactly the subset they use. Each consumer pays only for the methods they actually depend on.

**A concrete type is a superset of every possible interface over it.** You can always narrow a struct to an interface at the call site. You cannot widen an exported interface without breaking every implementor.

## 4. Producer-side interfaces create import cycles and dependency inversions in the wrong direction

When the implementing package exports the interface, every consumer must import the implementing package to name the interface — even consumers that never touch the implementation. A function signature like

```go
func Process(s storage.Storage) error
```

forces `Process`'s package to depend on `storage`, which may in turn pull in database drivers, network libraries, and configuration types — none of which `Process` actually uses.

Move the interface to the consumer and the direction flips:

```go
package process

type Store interface {
    Get(id string) ([]byte, error)
}

func Process(s Store) error { ... }
```

Now `process` depends on nothing. `storage.PostgresStorage` satisfies `process.Store` implicitly. The implementing package depends (transitively, through `main`) on the consumer, not the other way around. This is the dependency inversion principle, expressed through the type system without ceremony.

The practical consequence shows up in large codebases: packages that follow this convention form shallow, acyclic dependency graphs. Packages that don't tend to accumulate a central "types" or "interfaces" package that everything imports — a classic smell, and one that Go's tooling will eventually punish with build times and import cycles.

## 5. Testing gets simpler, not harder

The most common objection: "But if the implementing package doesn't export an interface, how do I mock it?"

You define the interface in your test, or in the consumer package. It is almost always smaller than whatever the producer would have exported:

```go
// in the consumer's test file
type fakeStore struct{ data map[string][]byte }

func (f *fakeStore) Get(id string) ([]byte, error) {
    return f.data[id], nil
}
```

Three lines. No mocking framework. No generated code. No dependency on the real `storage` package in your test binary. The fake satisfies the consumer's `Store` interface because structural typing does not care where either side was declared.

This is not a workaround — it is the point. Each test defines exactly the surface it needs to fake, and nothing more. Tests do not break when unrelated methods are added to the real implementation.

## 6. What the convention is not

A few clarifications, because this rule is often over-applied:

- **"Generally" means generally.** `io.Reader`, `io.Writer`, `error`, `fmt.Stringer`, `sort.Interface` — these live in packages that do not implement them precisely because they are consumer-side abstractions used by many consumers. When an interface is genuinely a shared vocabulary (not tied to a single implementation), a neutral package is the right home.
- **Plugin-style APIs are an exception.** If your package's *entire purpose* is to accept user-supplied implementations — `http.Handler`, `database/sql/driver.Driver` — the interface obviously belongs with the package that consumes it (which in these cases is the one you're writing). The rule still holds: interface with the consumer.
- **Small exported interfaces near implementations are occasionally fine.** If you truly have multiple implementations in the same package (say, three cache strategies) and callers legitimately choose between them at runtime, an interface there is reasonable. But verify that you actually have multiple implementations today — not a speculative second one that may arrive next quarter.

## Conclusion

"Accept interfaces, return structs" is not a taste preference. It is a direct consequence of four facts about Go:

1. Interfaces are satisfied structurally, so they don't need to live near implementations.
2. Consumers know their own requirements; producers can only guess.
3. Concrete return types preserve the freedom to add methods without breaking callers.
4. Consumer-side interfaces keep dependency graphs shallow and acyclic.

When you export an interface from the implementing package, you are paying the costs of nominal typing, over-specified contracts, breaking-change risk, and inverted dependencies — and getting, in return, a design that merely looks familiar to developers from other languages. Go gave you a better option. Use it.
