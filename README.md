# ironweave

A document graph engine with bidirectional links, versioning, and concurrent access.

> **Project 4 of 4** in the Rust learning series. This project teaches you every smart pointer type in Rust's standard library by building something real.

## What You're Building

A CLI tool and library for managing a graph of interconnected documents -- like Obsidian, Roam, or a personal wiki. Documents link to each other bidirectionally. You can query the graph, traverse connections, version documents with copy-on-write efficiency, and access the graph from multiple threads.

## Why This Project Teaches Smart Pointers

Every smart pointer in Rust exists to solve a specific problem. This project surfaces each problem naturally:

- **`Box<T>`**: Documents have nested sections (a recursive tree). You cannot size a recursive type without heap indirection.
- **`Rc<T>` + `Weak<T>`**: Documents link to each other. Forward links use strong references for shared ownership. Back-links use weak references to avoid reference cycles and memory leaks.
- **`RefCell<T>`**: You hold multiple shared handles to a document but need to edit its content. Interior mutability lets you mutate through shared references.
- **`Rc<RefCell<T>>`**: The classic combination -- shared ownership WITH mutation in single-threaded code.
- **`Arc<T>` + `Mutex<T>` / `RwLock<T>`**: Thread-safe equivalents of the above for concurrent access from a TCP server.
- **`Cow<'a, str>`**: Document fields that are usually borrowed from disk but sometimes owned after user edits. Copy-on-write avoids unnecessary allocations.
- **`Pin<T>`**: Async document loading where futures reference internal buffers. Self-referential async state must not be moved.
- **Custom smart pointer**: Build your own reference-counted handle with metadata tracking (access count, timestamps, labels).

## Architecture

```
+------------------------------------------+
|           CLI Interface                   |
|  ironweave new "My Document"             |
|  ironweave link doc1 doc2                |
|  ironweave query "linked-from:doc1"      |
|  ironweave serve --port 8080             |
+------------------+-----------------------+
                   |
+------------------v-----------------------+
|        Graph Engine                       |
|  +------------------------------------+  |
|  |  Documents (Rc<RefCell<Doc>>)      |  |
|  |  or Arc<RwLock<Doc>> for threads   |  |
|  +------------------+-----------------+  |
|  +------------------v-----------------+  |
|  |  Link Registry                     |  |
|  |  Forward: Rc<Doc>                  |  |
|  |  Backlink: Weak<Doc>              |  |
|  +------------------------------------+  |
|  +------------------------------------+  |
|  |  Version Store (Cow-based)         |  |
|  +------------------------------------+  |
+------------------+-----------------------+
                   |
+------------------v-----------------------+
|        Storage Layer                      |
|  File-based persistence (JSON/bincode)   |
+------------------------------------------+
```

## Prerequisites

**This is project 4 of 4.** You should have completed:

1. **rustcache** -- Ownership, borrowing, `&`/`&mut`/`*`, `ref` patterns, Entry API, interior mutability basics
2. **logforger** -- Lifetimes, `&str` vs `String`, multiple lifetimes, zero-copy patterns, `Cow`
3. **plugsmith** -- Traits, `Deref`/`DerefMut`, `AsRef`, trait objects, generics, `Fn` traits

Smart pointers combine ALL of these. If something feels shaky, revisit the relevant project first. This README will tell you exactly which prior project to revisit when a concept builds on previous work.

---

## Phase 1: Box -- Recursive Types and Heap Allocation

### Goal

Build the document model with nested sections. Understand why `Box` exists and when you need heap allocation.

### Rust Concepts Covered

- Stack vs heap: where data lives and why it matters for performance
- `Box<T>`: an owned pointer to heap-allocated data
- Recursive types: why the compiler cannot determine the size of a type that contains itself without indirection
- `Box` and `Deref`: you already learned `Deref` in plugsmith -- now see it powering `Box`'s transparent access to inner data
- `Box<dyn Trait>`: heap-allocated trait objects (you used these in plugsmith's pipeline -- now understand WHY they need `Box`)
- `Sized` vs `?Sized`: why trait objects and slices are "unsized" and require indirection

### What To Build

Build the following commands and data structures, using hints rather than code:

**Document and Section model:**
- Define a Document struct with an id, title, and content represented as a tree of Sections.
- A Section is recursive: it has text content and can contain child Sections (nested headings). Without `Box`, this type has infinite size -- the compiler will tell you exactly this. `Box` gives it a fixed size (one pointer width) because the child data lives on the heap.

**CLI commands:**
- **`ironweave new "Document Title"`** -- Creates a document with a root section. Think about how to generate unique IDs (UUIDs, incrementing counters, or slug from the title).
- **`ironweave edit <id>`** -- Add or edit sections using simple stdin-based input. No need for a full editor -- prompt the user for section title and content line by line.
- **`ironweave show <id>`** -- Render the document tree with indentation. Each nesting level adds visual indentation so the hierarchy is clear.
- **`ironweave outline <id>`** -- Show only headings (traverse the tree depth-first, skip leaf content). This forces you to write a recursive traversal over your boxed section tree.

**Key hints:**
- When traversing the section tree, you will have references to boxed sections. These auto-deref to references to the inner section via `Deref` (you learned this mechanism in plugsmith with newtypes -- same thing here).
- Try defining different section types (text, code block, quote) behind a common trait, then store them as boxed trait objects. This connects plugsmith's trait object work with the heap allocation that `Box` provides.
- Think about your Display implementations. You already know Display from plugsmith -- recursive Display on a tree structure is a good exercise.

### Questions To Answer Before Coding

1. What is the difference between stack and heap allocation? Why is stack allocation faster?
2. Why can the compiler not determine the size of a recursive enum without `Box`? Draw the memory layout on paper.
3. If `Box<T>` implements `Deref` targeting `T`, what does that mean for method calls? (Recall your `Deref` work from plugsmith.)
4. What is the difference between `Box<T>` and just `T`? When does `Box` add value beyond enabling recursion?
5. What is `?Sized` and why does `Box<dyn Trait>` require the trait to be unsized?
6. When you drop a `Box<T>`, what happens to the heap memory? Who is responsible?

### End Expectations

- Document tree renders correctly with nested sections
- Your recursive section type compiles (it uses `Box` for child sections)
- You can create documents with 5+ levels of nesting
- `ironweave outline` traverses the tree depth-first and prints only headings
- You can draw the memory layout of your document on paper: what lives on the stack, what lives on the heap, where each `Box` points

---

## Phase 2: Rc and Weak -- Shared Ownership and Breaking Cycles

### Goal

Add links between documents. Multiple documents reference the same target. Back-links exist without creating reference cycles.

### Rust Concepts Covered

- `Rc<T>`: reference-counted shared ownership (single-threaded only)
- Reference counting: how `Rc` tracks owners and drops the inner value when the count reaches zero
- `Rc::clone` -- cheap (increments a counter), NOT a deep copy. You learned `Clone` vs `Copy` in rustcache -- `Rc::clone` is semantically different from both.
- `Weak<T>`: a non-owning reference that does not prevent deallocation
- Reference cycles: why two `Rc` values pointing at each other creates a memory leak. Rust prevents data races, NOT reference cycle leaks.
- `Weak::upgrade()` -- returns an `Option`, which is `None` if the target has been dropped
- When to use `Rc` vs when to use plain references with lifetimes (your logforger knowledge). `Rc` is for when you genuinely cannot express the ownership lifetime.

### What To Build

**Graph structure:**
- Create a Graph struct that stores documents. Each document is wrapped so that multiple parts of the graph can hold a handle to the same document via shared ownership.

**CLI commands:**
- **`ironweave link <source-id> <target-id>`** -- Creates a link from source to target. The source document holds a strong shared reference to the target. This is shared ownership: both the graph's registry AND the source document "own" the target.
- **`ironweave backlinks <id>`** -- Show all documents that link TO this document. You will call `upgrade()` on each weak reference. Some might return `None` if the source document has been deleted since the link was created.
- **`ironweave unlink <source-id> <target-id>`** -- Remove a link. Watch the strong reference count decrease.
- **`ironweave delete <id>`** -- Delete a document. All strong references from other documents lose one count. All weak back-links become `None` on upgrade. No dangling pointers. No crashes. This is the safety guarantee.
- **`ironweave info <id>`** -- Show the document's strong count and weak count. This command helps you visualize the reference counting in action.
- **`ironweave reach <from> <to>`** -- Find a path between two documents by traversing links. This is graph traversal (BFS or DFS) over your shared-ownership structure.

**Key hints:**
- Forward links (source to target) should use strong shared references. Back-links (target back to source) should use weak references. WHY weak? If both directions used strong references, you would create a cycle (A points to B, B points to A) and neither would ever be dropped. Weak breaks the cycle because it does not contribute to the strong count.
- Try creating a cycle ON PURPOSE: make A link to B and B link to A, both with strong references. Then delete both from the graph's top-level registry. They will NOT be freed -- this is a memory leak in safe Rust. Now fix it by using weak references for back-links and verify the leak disappears.

### Questions To Answer Before Coding

1. How does reference counting work? What are strong count and weak count?
2. Why is `Rc::clone` cheap? What does it actually do vs what `Clone` usually does? (Recall `Clone` from rustcache.)
3. What happens when you call `Weak::upgrade()` and the target has been dropped? How should you handle this in your code?
4. Why can `Rc` not prevent reference cycle memory leaks? What would it need to detect cycles?
5. Why is `Rc` neither `Send` nor `Sync`? What happens if two threads try to increment the reference count simultaneously? (Recall `Send`/`Sync` from rustcache Phase 5.)
6. When should you use `Rc<T>` vs a reference `&T`? When can lifetimes (logforger) not solve the problem?

### End Expectations

- `ironweave link A B` then `ironweave backlinks B` shows A
- Deleting a document does not crash -- weak references gracefully become `None`
- `ironweave info <id>` shows correct strong and weak counts
- No memory leaks: you can demonstrate the cycle problem and the weak-reference fix
- `ironweave reach <from> <to>` finds a path between connected documents
- You understand WHEN shared ownership is needed (when you cannot express the lifetime) vs when a simple reference suffices

---

## Phase 3: RefCell and the Rc+RefCell Pattern -- Interior Mutability

### Goal

Edit documents through shared handles. Deeply understand interior mutability and the runtime borrow checker.

### Rust Concepts Covered

- `RefCell<T>`: moves borrow checking from compile-time to runtime
- `borrow()` and `borrow_mut()` -- the runtime equivalents of `&` and `&mut`
- Runtime panic on double mutable borrow (vs compile-time error with regular references)
- The classic combined pattern: shared ownership plus interior mutation for single-threaded code
- Comparison: `Cell<T>` vs `RefCell<T>` (you used `Cell` for counters in rustcache Phase 3 -- `RefCell` is for non-Copy types)
- `Ref<T>` and `RefMut<T>`: the guard types returned by borrow and borrow_mut
- Borrow guard scoping: a mutable guard must be dropped before you can take a shared borrow again

### What To Build

**Interior mutability integration:**
- Wrap each document so that any handle can both read AND modify the document. The combination of shared ownership and interior mutability makes this possible.

**CLI commands:**
- **`ironweave edit <id> --title "New Title"`** -- Find the document by id (which gives you a shared-ownership handle), then use mutable borrowing to change the title. Other handles to the same document see the change immediately.
- **`ironweave append <id> "New content"`** -- Mutate content through a shared reference. This is what interior mutability enables: you have a shared reference (from the shared-ownership layer's `Deref`) but can still modify the inner data (via the interior mutability layer's mutable borrow).
- **`ironweave diff <id>`** -- Compare current content with last-saved content. You need to borrow the document (shared) and read from storage simultaneously. Think about whether runtime borrow tracking interferes with file I/O (it should not -- file I/O does not go through the borrow guard).

**Key hints:**
- Implement a `modified` timestamp that updates whenever the document is edited. Be careful about borrow scoping: if you hold a shared borrow guard and try to take a mutable borrow guard in the same scope, it panics at runtime. You must drop the shared guard first.
- Build a `Transaction` struct that borrows a document mutably for a series of edits, then releases the borrow when it is dropped. This forces you to think about guard lifetimes -- which you practiced in logforger.
- Try intentionally causing a runtime borrow panic: hold a mutable guard while calling for a shared borrow. Then fix it. Understand that this panic is a RUNTIME version of the compile-time borrow error you encountered in rustcache.
- Write a test that intentionally triggers the panic and catches it, proving you understand exactly when it occurs.

### Questions To Answer Before Coding

1. What does `RefCell<T>` allow that regular `&mut T` does not? What is the trade-off?
2. What happens if you call `borrow_mut()` while a `borrow()` guard is still alive? Why is this a panic rather than a compile error?
3. What is `Ref<T>` (the return type of `borrow()`)? Why does it not just return a plain `&T`? (Hint: who tracks when the borrow ends?)
4. Why is the combined shared-ownership + interior-mutability pattern so common? What specific problem does each layer solve?
5. How does `RefCell` compare to `Cell`? When would you use `Cell` (rustcache Phase 3) vs `RefCell`?
6. If `RefCell` panics on double-borrow at runtime, have you lost Rust's safety guarantees? What guarantee do you still have?

### End Expectations

- You can edit documents through any shared handle; changes are visible from all handles immediately
- `ironweave edit <id> --title "New"` modifies the document without cloning it
- No runtime borrow panics in normal usage (all borrow guards are properly scoped)
- Transaction struct groups edits safely with a scoped mutable guard
- You can explain the difference between compile-time borrow checking (your rustcache experience) and runtime borrow checking
- A test intentionally triggers a `RefCell` panic and catches it to prove you understand the failure mode

---

## Phase 4: Cow -- Copy-on-Write Efficiency

### Goal

Add document versioning with copy-on-write semantics. Most versions share content; only modified portions get cloned.

### Rust Concepts Covered

- `Cow<'a, str>` and `Cow<'a, [T]>`: borrowed by default, owned when modified
- `Borrowed` vs `Owned` variants -- the enum that decides at RUNTIME whether data is borrowed or owned
- `to_mut()` -- the copy-on-write trigger: clones borrowed data into owned form only when you first mutate
- `Cow` implements `Deref` -- so you can use it anywhere a plain reference is expected (the `Deref` pattern from plugsmith yet again)
- Accepting both borrowed and owned strings ergonomically via conversion traits
- When `Cow` makes sense: read-mostly data that is rarely modified (exactly like document versions)
- Lifetimes with `Cow`: the lifetime parameter is the lifetime of the borrowed variant. Your logforger lifetime knowledge applies directly.

### What To Build

**Versioning system:**
- When you create a new version of a document, fields that have not changed remain borrowed from the previous version. Fields that have changed become owned. This is copy-on-write at the field level.

**CLI commands:**
- **`ironweave history <id>`** -- Show all versions of a document. Most versions share the same memory for unchanged fields.
- **`ironweave checkout <id> <version>`** -- Load a historical version of a document.
- **`ironweave edit <id> --title "New Title"`** -- Creates a new version. The title field triggers a clone-on-write (borrowed becomes owned). All other fields stay borrowed from the previous version. Memory efficient.
- **`ironweave diff <id> <v1> <v2>`** -- Compare two versions. Because `Cow` implements `Deref` to the underlying string type, comparison works regardless of whether versions are borrowed or owned.

**Key hints:**
- Think about how the lifetime parameter ties the borrowed variant to the source data. If the source version gets dropped, any borrowed cow pointing to it would dangle -- but the lifetime system prevents this. You saw this exact concept in logforger (data cannot outlive its source). If you need the version to outlive the source, convert it to owned.
- Build a template system: `ironweave template "Hello, {name}!"` -- the template string is usually borrowed but when variables are substituted, only those portions become owned. This is a classic use case for copy-on-write strings.
- Benchmark your copy-on-write versioning against a naive approach that clones everything. Measure memory usage with 100 versions of a large document where each version changes only one field.

### Questions To Answer Before Coding

1. What is the memory difference between the borrowed variant holding a string slice and the owned variant holding a heap-allocated string?
2. When does `to_mut()` actually clone? What if it is already owned?
3. How does `Cow` implement `Deref`? Why does this matter for using it with APIs that expect references?
4. What happens to the lifetime when you convert to owned? Why does the resulting type have no lifetime parameter?
5. When would you use `Cow` vs just cloning everything? What is the performance trade-off?
6. How does `Cow` relate to the zero-copy parsing you did in logforger? Could logforger's entry struct use `Cow` instead of plain string slices?

### End Expectations

- `ironweave history <id>` shows versions efficiently with shared memory for unchanged fields
- Creating a new version with one field changed does not clone the entire document
- Template substitution works with copy-on-write strings
- Memory usage: 100 versions of a large document with small changes uses significantly less memory than 100 full copies
- You can explain where each `Cow` is borrowed vs owned in your data structures
- Benchmark results comparing copy-on-write versioning vs clone-everything versioning

---

## Phase 5: Arc, Mutex, RwLock -- Thread-Safe Smart Pointers

### Goal

Make the document graph safe for concurrent access. Understand the thread-safe equivalents of the single-threaded smart pointers.

### Rust Concepts Covered

- `Arc<T>`: atomic reference counting (thread-safe shared ownership)
- Why `Rc` is not `Send` or `Sync` (non-atomic counter means data races)
- `Arc<Mutex<T>>`: thread-safe shared mutable state (the pattern you briefly used in rustcache Phase 5 -- now deeply understand it)
- `Arc<RwLock<T>>`: readers do not block each other (better for read-heavy document workloads)
- `MutexGuard`, `RwLockReadGuard`, `RwLockWriteGuard`: RAII lock guards (they implement `Deref` and `DerefMut` -- recognizing the pattern from plugsmith)
- Lock poisoning: what happens when a thread panics while holding a lock
- Deadlocks: not prevented by the compiler, still your responsibility
- The `parking_lot` crate as an alternative: faster locks without poisoning

### What To Build

**Thread-safe graph engine:**
- Replace the single-threaded shared-ownership + interior-mutability wrappers with their thread-safe equivalents throughout the graph engine.

**CLI commands and server:**
- **`ironweave serve --port 8080`** -- A simple TCP server. Multiple clients connect and read/write documents concurrently.
- Multiple readers: several clients running `show <id>` simultaneously. Read-write locks allow this without blocking.
- Writer exclusivity: one client runs `edit <id>`. The write lock gives exclusive access, blocking readers until the edit is complete.
- **`ironweave lock <id>`** / **`ironweave unlock <id>`** -- Application-level locking for long editing sessions, separate from the mechanical locking of mutexes and read-write locks.

**Key hints:**
- Graph traversal under concurrency: `ironweave reach <from> <to>` acquires read locks on each document in the path. Think about lock ordering to avoid deadlocks. If an operation needs to lock two documents (such as linking), always acquire locks in a consistent order (for example, by document ID).
- Implement a `--readonly` flag for clients: if set, only read locks are acquired. The type system should enforce this (consider using a trait or a newtype -- apply your plugsmith trait knowledge).
- Compare performance: `Mutex` vs `RwLock` for your workload. Read-heavy (many shows, few edits) vs write-heavy (many edits). Benchmark both.
- Sharded locking: instead of one big lock on the whole graph, lock individual documents. What happens when an operation needs to lock TWO documents at once? Lock ordering prevents deadlocks.

### Questions To Answer Before Coding

1. Why does `Rc` use a regular integer for its counter while `Arc` uses an atomic integer? What is the cost of atomics?
2. You used the atomic-shared-mutex pattern in rustcache. Now explain: what exactly happens when two threads call `.lock()` simultaneously?
3. What is lock poisoning? Should you use `.lock().unwrap()` or handle the poisoned case?
4. How can deadlocks happen with multiple locks? How does consistent lock ordering prevent them?
5. `MutexGuard` implements `Deref` -- how does this relate to the `Deref` you implemented in plugsmith?
6. When would you use a mutex vs a read-write lock? What is the overhead difference?

### End Expectations

- Multiple TCP clients reading documents concurrently without blocking each other
- Writes are exclusive and consistent
- No deadlocks (lock ordering is implemented for multi-document operations)
- Benchmark shows read-write lock advantage for read-heavy workloads
- `ironweave serve` handles 10+ concurrent connections
- You can explain every thread-safe smart pointer in your code and why it is needed there vs the single-threaded equivalent

---

## Phase 6: Pin, Custom Smart Pointers, and Production Polish

### Goal

Handle self-referential async patterns with `Pin`. Build a custom smart pointer. Polish the project for production quality.

### Rust Concepts Covered

- `Pin<Box<T>>` and `Pin<&mut T>`: preventing moves after construction
- Why moves break self-referential types (a struct with a field that points to another field -- moving the struct invalidates the internal pointer)
- `Unpin` trait: most types are `Unpin` (can be freely moved). Only self-referential types (like async futures) need pinning.
- Async futures are self-referential -- that is why `.await` interacts with `Pin`
- Custom smart pointer: implement `Deref`, `DerefMut`, and `Drop` on your own type (using everything from plugsmith's trait knowledge)
- `Drop` trait: custom cleanup logic when values go out of scope
- `std::mem::drop` vs the `Drop` trait implementation
- Phantom types for compile-time state tracking

### What To Build

**Async document loading:**
- **`ironweave load <path> --async`** -- Uses async I/O to load large documents. The async state machine (the Future) is self-referential: it holds a buffer AND a reference into that buffer. `Pin` ensures it does not move and invalidate that reference. You do not need to implement `Pin` manually -- just understand WHY async runtimes require pinned futures when you use them.

**Custom smart pointer:**
- Build a `TrackedHandle<T>` -- like a reference-counted pointer but it also tracks: creation time, last access time, access count, and a creator label (a string identifying who created the handle).
  - Implement `Deref` targeting `T` so the handle auto-derefs to the inner value.
  - Implement `DerefMut` so mutable access works transparently.
  - Implement `Drop` to log when the handle is freed. This is how you would build debugging and profiling tools.
  - Implement `Clone` -- each clone increments the shared access count.
  - Use this handle in your graph to track which documents are most accessed.
  - **`ironweave hottest`** -- Shows the top 10 most-accessed documents based on handle metrics.

**Lazy loading:**
- Implement a `DocumentHandle` that can be in two states: `Loaded` (content in memory) and `Unloaded` (only metadata, content on disk). Use phantom types or an enum. Loading transitions from unloaded to loaded. Design question: should the handle implement `Deref` to auto-load on access? That would be convenient but surprising (magic). Or should loading be explicit? Make a deliberate design choice and document your reasoning.

**Production polish:**
- Proper error handling with `thiserror` and `anyhow` (revisit from plugsmith Phase 6)
- CLI with `clap` derive macros
- File-based persistence: save and load the graph to JSON or bincode
- Import from Markdown files with `[[wiki-link]]` syntax
- Export graph as DOT format for visualization: `ironweave export --format dot | dot -Tpng -o graph.png`
- Comprehensive test suite including concurrency tests

### Questions To Answer Before Coding

1. What does "moving" a value mean in terms of memory? Why does moving break a self-referential type?
2. What does `Pin<Box<T>>` guarantee? What can and what cannot you do with a pinned value?
3. What does the `Unpin` trait mean? Why are most types automatically `Unpin`?
4. When you implement `Deref` for your custom handle, what does the `Target` type need to be?
5. When you implement `Drop` for your custom handle, what can you do in the drop function? What are you NOT allowed to do?
6. How does implementing `Deref` + `DerefMut` + `Drop` + `Clone` make your custom type behave like a smart pointer?

### End Expectations

- Async document loading works with pinned futures
- `TrackedHandle` works as a drop-in replacement for standard shared-ownership pointers in the graph
- `ironweave hottest` shows most-accessed documents using handle metrics
- `ironweave export --format dot | dot -Tpng -o graph.png` generates a visual graph
- File persistence: documents survive process restart
- Comprehensive test suite including concurrency tests
- You can implement a custom smart pointer from scratch and explain every trait it needs

---

## Project Structure

```
ironweave/
├── Cargo.toml
├── README.md
├── src/
│   ├── main.rs              # CLI entry point
│   ├── lib.rs               # Public API
│   ├── document.rs          # Document struct (Box for recursive sections)
│   ├── section.rs           # Section tree (recursive with Box)
│   ├── graph.rs             # Document graph (Rc/Arc + links)
│   ├── link.rs              # Link types (forward Rc, backward Weak)
│   ├── version.rs           # Version history (Cow-based)
│   ├── handle.rs            # TrackedHandle (custom smart pointer)
│   ├── server.rs            # TCP server (Phase 5)
│   ├── storage.rs           # File persistence
│   ├── query.rs             # Graph queries and traversal
│   ├── export.rs            # DOT/Graphviz export
│   ├── config.rs            # Configuration
│   └── error.rs             # Error types
├── tests/
│   ├── document_tests.rs
│   ├── graph_tests.rs
│   ├── version_tests.rs
│   ├── concurrency_tests.rs
│   └── handle_tests.rs
└── benches/
    ├── cow_vs_clone.rs      # Cow versioning vs full clone
    ├── mutex_vs_rwlock.rs   # Lock strategy comparison
    └── graph_traversal.rs
```

## The Smart Pointer Journey

| Phase | The Moment It Clicks | Smart Pointer |
|-------|---------------------|---------------|
| 1 | "My recursive type has infinite size -- Box puts it on the heap with a fixed-size pointer" | `Box<T>` |
| 2 | "Two documents both 'own' the same target -- Rc lets them share. Weak breaks the cycle." | `Rc<T>`, `Weak<T>` |
| 3 | "I have an Rc handle but need to modify the document -- RefCell checks borrows at runtime" | `RefCell<T>`, `Rc<RefCell<T>>` |
| 4 | "Most versions share content. Only modified parts get cloned. Cow decides at runtime." | `Cow<'a, T>` |
| 5 | "Rc panics across threads. Arc uses atomics. Mutex and RwLock replace RefCell." | `Arc<T>`, `Mutex<T>`, `RwLock<T>` |
| 6 | "Async futures cannot be moved because they reference themselves. Pin prevents the move." | `Pin<T>`, Custom |

## The Smart Pointer Decision Tree

When you need to choose a pointer, ask these questions in order:

1. **Do you need heap allocation for a recursive or unsized type?** Use `Box<T>`.
2. **Do you need shared ownership (multiple owners)?**
   - Single thread: `Rc<T>`
   - Multiple threads: `Arc<T>`
3. **Do you need to mutate through shared ownership?**
   - Single thread: combine shared ownership with `RefCell<T>`
   - Multiple threads, any access pattern: combine atomic shared ownership with `Mutex<T>`
   - Multiple threads, read-heavy: combine atomic shared ownership with `RwLock<T>`
4. **Do you need "usually borrowed, sometimes owned"?** Use `Cow<'a, T>`.
5. **Do you need to prevent moves (self-referential types)?** Use `Pin<T>`.
6. **Can you just use a reference with a lifetime?** DO THAT FIRST.

Rule 6 is the most important. Smart pointers are a tool of LAST RESORT after references and lifetimes. If logforger taught you lifetimes well, you will reach for smart pointers less often -- and that is the right instinct.

## Common Pitfalls

1. **"cannot move out of Rc"** -- Shared ownership gives you shared access, not ownership of the inner value. Use `try_unwrap()` to extract the inner value if there is exactly one strong reference remaining, or clone the contents.

2. **"already borrowed: BorrowError" (runtime panic)** -- You called `borrow()` while a `borrow_mut()` guard was still alive. Scope your guards carefully. This is the runtime version of the compile-time error you encountered in rustcache.

3. **Memory leak with reference cycles** -- If A holds a strong reference to B and B holds a strong reference to A, neither is ever freed. Use weak references for back-links. The `ironweave info` command helps visualize this.

4. **"Rc is not Send"** -- You tried to share a single-threaded reference-counted pointer across threads. Use the atomic variant instead. The compiler catches this because the single-threaded version does not implement `Send` (recall `Send`/`Sync` from rustcache Phase 5).

5. **Deadlock with multiple locks** -- Thread 1 locks A then B. Thread 2 locks B then A. Both wait forever. Always acquire locks in a consistent order (for example, by document ID).

6. **Overusing smart pointers** -- Before reaching for shared ownership or atomic pointers, ask: can I restructure to use plain references with lifetimes? (Your logforger skills.) Smart pointers have runtime cost -- reference counting, lock overhead, cache misses from indirection. Use them when lifetimes genuinely cannot express the relationship.

## How Prior Projects Feed Into This One

| Concept | Where You Learned It | How It Applies Here |
|---------|---------------------|---------------------|
| Ownership, `&`, `&mut`, `*` | rustcache Phase 1-2 | Every smart pointer derefs -- you need fluency with references |
| Entry API, interior mutability | rustcache Phase 2-3 | `RefCell` IS interior mutability. Same concept, different mechanism. |
| `Cell<T>` for counters | rustcache Phase 3 | `Cell` for TrackedHandle's access count (Copy types through shared ref) |
| `Send`/`Sync`, `Arc`, `Mutex` | rustcache Phase 5 | Phase 5 builds directly on this. Now you understand WHY. |
| Lifetimes, string borrowing | logforger Phase 1-2 | `Cow` has a lifetime -- it is borrowed data with an escape hatch to ownership |
| Zero-copy patterns | logforger Phase 3-4 | `Cow` is the pragmatic middle ground between zero-copy and fully owned |
| `Pin`, self-referential problems | logforger Phase 4 | You encountered the lending iterator problem. `Pin` is the deeper explanation. |
| `Deref`/`DerefMut` traits | plugsmith Phase 4 | `Box`, `Rc`, `Arc`, lock guards ALL use `Deref`. Your custom pointer implements it. |
| Trait objects, boxed traits | plugsmith Phase 2 | Boxed trait objects = heap-allocated trait objects. Now you understand the `Box` part. |
| `Display`, `Debug`, `Drop` traits | plugsmith Phase 6 | Custom smart pointer needs `Drop` for cleanup, `Display` for debugging |
| `AsRef`, `Borrow` | plugsmith Phase 4 | `Cow` implements these. They are how smart pointers compose with generic APIs. |

## Key Dependencies

```toml
# Phases 1-4: zero external dependencies -- all smart pointers are in std
[dependencies]

# Phase 5 (optional)
# parking_lot = "0.12"  # faster Mutex/RwLock without poisoning

# Phase 6
tokio = { version = "1", features = ["full"] }
clap = { version = "4", features = ["derive"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
thiserror = "2"
anyhow = "1"
tracing = "0.1"
tracing-subscriber = "0.3"
```

## Success Criteria

By completing this project you will:

- Know every smart pointer in the standard library and when to use each one
- Understand the decision tree: reference, then Box, then Rc/Arc, then RefCell/Mutex, then Cow, then Pin
- Build data structures with shared ownership (graphs, trees with back-references)
- Handle interior mutability without runtime panics
- Write thread-safe concurrent code with proper locking strategies
- Implement your own smart pointer with `Deref`, `DerefMut`, `Drop`, and `Clone`
- Understand `Pin` well enough to work with async Rust confidently
- Know when NOT to use smart pointers -- reach for `&T` first, smart pointers only when lifetimes cannot express the relationship
