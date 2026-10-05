# PMR Arena Allocator

A fixed-capacity bump (arena) allocator in C++20 that plugs into `std::pmr`. Allocation is a pointer bump, individual frees are no-ops, and everything is reclaimed at once with `reset()`. Objects created through the arena have their destructors run automatically on `reset()` or when the arena is destroyed.

Built as a learning project: written from scratch, then debugged and verified with AddressSanitizer and UBSan.

## Features

- Derives from `std::pmr::memory_resource`, so it works with `std::pmr::vector`, `std::pmr::string`, and other PMR containers
- Alignment-aware allocation; padding is counted in the bounds check
- Throws `std::bad_alloc` when the block is exhausted
- `create<T>(args...)` constructs an object inside the arena, forwarding constructor arguments
- Type-erased destructor list: each non-trivially-destructible object is registered with a captureless lambda that calls `std::destroy_at<T>`
- `reset()` destroys registered objects and rewinds the arena for reuse

## Usage

```cpp
memory<int> arena(1024);

// Pass the arena as the PMR resource so containers allocate from it
auto *v = arena.create<std::pmr::vector<std::pmr::string>>(&arena);
v->push_back("hello");

arena.reset();   // destroys registered objects, rewinds the pointer
```

Use `std::pmr::string` (not `std::string`) for elements, otherwise the string buffers come from the global heap instead of the arena.

## Build and test

```bash
g++ -std=c++20 -Wall -Wextra -g -fsanitize=address,undefined malloc_pmr.cpp -o arena_test
./arena_test
```

A silent run with exit code 0 means all tests passed and the sanitizers found nothing. (In fish, check with `echo $status`.)

### What the tests cover

| Test | What it checks |
| --- | --- |
| Alignment | A pointer allocated after an odd-sized allocation is correctly aligned |
| Padding in bounds check | An allocation whose alignment padding would overflow the block throws instead of overrunning |
| Destructors run once | Each registered destructor runs exactly once across repeated `reset()` calls |
| PMR wiring | A `pmr::vector`'s buffer and its `pmr::string` elements live inside the arena |

The tests were also checked to fail when the code is deliberately broken (for example, removing the bounds check), so a clean run is meaningful.

## Design notes

- **Why destruction is type-erased:** the arena stores `{object pointer, destroy function}` pairs. The type is only known inside `create<T>()`, so the lambda captures it there and the arena later calls it without knowing `T`.
- **Why fully-PMR containers don't need registration:** a `pmr::vector` frees its own elements, and with a no-op `do_deallocate` all of its memory lives in the arena anyway. Registering it as well risks double destruction.
- **Alignment waste:** padding before aligned allocations is never reused (the pointer only moves forward). The waste is at most `align - 1` bytes per allocation and shrinks if you allocate in descending alignment order.

## Limitations

- Fixed capacity: no chaining of additional blocks when full
- Not thread-safe
- No individual deallocation; memory is only reclaimed by `reset()` or destruction
- Not copyable or movable

## Possible next steps

- Chain multiple blocks instead of throwing when one fills up (compare with `std::pmr::monotonic_buffer_resource`)
- Track and report padding overhead
- Remove the unused class template parameter
