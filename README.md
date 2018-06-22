# process-lib

## Design

process-lib is designed to be a minimal wrapper around the platform-specific
API's for starting a process, interacting with its standard streams and finally
terminating it.

### Opaque pointer

process-lib uses a process struct to store information between calls to library
functions. This struct is forward-declared in process.h with each
platform-specific implementation hidden in the source files for that platform.

This struct is typedefed to Process and exposed to the user as an opaque pointer
(Process \*).

```c
typedef struct process Process;
```

The process.h header only contains a forward-declaration of the process struct.
We provide an implementation in the source files of each supported platform
where we store platform-specific members such as pipe handles, process id's,
....

Advantages:

- Fewer includes required in process.h

  Because we only have a forward declaration of the process struct in process.h
  we don't need any platform-specific includes (such as windows.h) in the header
  file to define the data types of all the members that the struct contains.

- No leaking of implementation details

  Including the process.h header only gives the user access to the forward
  declaration of the process struct and not its implementation which means its
  impossible to access the internals of the struct outside of the library. This
  allows us to change its implementation between versions without having to
  worry about breaking user code.

Disadvantages:

- No simple allocation on the stack

  Because including process.h does not give the compiler access to the full
  definition of the process struct it is unable to allocate its implementation
  on the stack since it doesn't know its size. Allocating on the stack is still
  possible but requires functions such as
  [alloca](https://linux.die.net/man/3/alloca) which is harder compared to just
  writing

  ```c
  Process process;
  ```

- Not possible to allocate on the heap without help from the library

  Because the compiler doesn't know the size of the process struct the user
  can't easily allocate the process struct on the heap because it can't figure
  out its size with sizeof.

  Because the library has to allocate memory regardless (see
  [Memory Allocation](#memory-allocation)), this problem is solved by providing
  the `process_alloc` function which allocates the required memory returns the
  resulting pointer to the user. If no memory allocations were required in the
  rest of the library, `process_alloc` would likely be replaced with a
  `process_size` function which would allow the user to allocate the required
  memory himself and allow the library code to be completely free of dynamic
  memory allocations.

### Memory allocation

process-lib aims to do as few dynamic memory allocations as possible. As of this
moment, memory allocation is only done when allocating memory for the process
struct in `process_alloc` and when converting the array of program arguments to
a single string as required by the Windows
[CreateProcess](<https://msdn.microsoft.com/en-us/library/windows/desktop/ms682425(v=vs.85).aspx>)
function.

I have not found a way to avoid allocating memory while keeping a cross-platform
api for POSIX and Windows. (Windows `CreateProcessW` requires a single string of
arguments delimited by spaces while POSIX `execvp` requires an array of
arguments). Since process-lib's api takes process arguments as an array we have
to allocate memory to convert the array into a single string as required by
Windows.

process-lib uses the standard `malloc` and `free` functions to allocate and free
memory. However, providing support for custom allocators should be
straightforward. If you need them, please open an issue.

### (Posix) Waiting on process with timeout

I did not find a counterpart for the Windows
[WaitForSingleObject](<https://msdn.microsoft.com/en-us/library/windows/desktop/ms687032(v=vs.85).aspx>)
function which can be used to wait until a process exits or the provided timeout
expires. POSIX has a similar function called
[waitpid](https://linux.die.net/man/2/waitpid) but this function does not
support specifying a timeout value.

To support waiting with a timeout value on POSIX, each process is put in its own
process group with the same id as the process id with a call to `setpgid` after
forking the process. When calling the `process_wait` function, a timeout process
is forked which we put in the same process group as the process we want to wait
for with the same `setpgid` function and puts itself to sleep for the requested
amount of time (timeout value) before exiting. We then call the `waitpid`
function in the main process but instead of passing the process id of the
process we want to wait for we pass the negative value of the process id
`-process->pid`. Passing a negative value for the process id to `waitpid`
instructs it to wait for all processes in the process group of the absolute
value of the passed negative value. In our case it will wait for both the
timeout process we started and the process we actually want to wait for. If
`waitpid` returns the process id of the timeout process we know the timeout
value has been exceeded. If `waitpid` returns the process id of the process we
want to wait for we know it has exited before the timeout process and that the
timeout value has not been exceeded.

This solution was inspired by [this](https://stackoverflow.com/a/8020324)
Stackoverflow answer.
