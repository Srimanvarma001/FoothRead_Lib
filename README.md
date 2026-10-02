# Foothread

A small user-level threading library for Linux, written in C. It's a pthreads-style API built directly on the `clone()` system call, with mutexes and barriers implemented on System V semaphores.

I built it for the Operating Systems course at IIT Kharagpur.

## Features

- **Thread creation** with `clone()` (`CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SIGHAND | CLONE_THREAD`), so threads share memory, file descriptors and signal handlers
- **Joinable and detached threads**, set through `foothread_attr_t`
- **Custom stack sizes** per thread (default 2 MB)
- **Mutexes** (`foothread_mutex_*`) with an owner check on unlock
- **Barriers** (`foothread_barrier_*`) for N threads, reusable across rounds
- Builds as a **shared library**, `libfoothread.so`

## API

```c
void foothread_create(foothread_t *, foothread_attr_t *, int (*)(void *), void *);
void foothread_exit();
void foothread_attr_setjointype(foothread_attr_t *, int);   // FOOTHREAD_JOINABLE / FOOTHREAD_DETACHED
void foothread_attr_setstacksize(foothread_attr_t *, int);

void foothread_mutex_init(foothread_mutex_t *);
void foothread_mutex_lock(foothread_mutex_t *);
void foothread_mutex_unlock(foothread_mutex_t *);
void foothread_mutex_destroy(foothread_mutex_t *);

void foothread_barrier_init(foothread_barrier_t *, int count);
void foothread_barrier_wait(foothread_barrier_t *);
void foothread_barrier_destroy(foothread_barrier_t *);
```

Passing `NULL` as the attribute to `foothread_create` creates a detached thread with the default stack size.

### Joining

There is no separate `join` call. A thread that calls `foothread_exit()` first waits for every **joinable** thread it created, then marks itself finished. So the main thread calling `foothread_exit()` at the end of `main` waits for all its joinable children.

## Demo: parallel tree sum

`computesum.c` uses the library to add up values over a tree, with **one thread per node**:

1. `gentree` writes a random tree to `tree.txt`.
2. `computesum` reads the tree and asks for a value for each leaf node.
3. Each internal node's thread waits on a barrier until all of its children have added their subtree sums, then passes its own sum to its parent. A mutex protects the shared sums.
4. The root prints the total.

`tree.txt` format: the first line is the number of nodes `n`, then one line per node with `node parent`. The root is its own parent.

## Build and run

You need Linux and gcc (glibc 2.30 or newer, for `gettid()`).

```bash
make lib        # builds libfoothread.so
make app        # builds the library and the computesum demo
make tree       # builds the gentree tree generator

make newrun     # generate a random 25-node tree, then run the demo
make run        # run the demo on the existing tree.txt

./gentree 10    # generate a tree with 10 nodes instead
make clean
```

Example run on a 6-node tree (this random tree had 4 leaves):

```
$ ./gentree 6 && printf '1\n2\n3\n4\n' | ./computesum
Reading values for leaf nodes:
Enter value for node 2: Node 2 value: 1
...
SOL => Total sum = 10
```

## Project structure

| File | What it is |
|---|---|
| `foothread.h` | Public API, types and constants |
| `foothread.c` | Library implementation |
| `computesum.c` | Tree-sum demo application |
| `gentree.c` | Random tree generator |
| `makefile` | Build targets |

## Limitations

This is a course project for learning, not a production library.

- At most `FOOTHREAD_THREADS_MAX` (100) threads per process.
- Thread stacks are allocated with `malloc` and are not freed when a thread exits.
- Mutex and barrier semaphores are created with `ftok(".", n)` keys, so they are shared by anything running in the same directory. If a run is interrupted, they stay in the system. List them with `ipcs -s` and remove them with `ipcrm -s <id>`.
