## Q\&A

### Q: How are the commands (user-space/userland) executed ?

Once your HTTP handler has parsed and validated the incoming JSON `{ cmd: "...", args: [...] }`, here’s the usual sequence inside your daemon to actually “run” that command and capture its output:

```
[Daemon HTTP Handler]
             │
    1. Validate command (allow-list, sanitize args)
             │
    2. Create two anonymous pipes:
       ┌────────────┐    ┌────────────┐
       │ stdoutPipe │    │ stderrPipe │
       │ read end   │◀───┐ write end  │
       └────────────┘    └────────────┘
             │                  │
             │ pipe()           │ pipe()
             │                  │
             ▼                  ▼
     ┌─────────────────────────────────┐
     │ fork() → child PID              │
     └─────────────────────────────────┘
             │
   ┌─────────┴───────────┐
   │                     │
   ▼                     ▼
[ Child Process ]   [ Parent (Daemon) ]
   • dup2(stdoutPipe.write, STDOUT_FILENO)  
   • dup2(stderrPipe.write, STDERR_FILENO)  
   • close	unused pipe ends
   • execvp(cmd, argv)   ← replaces child with the requested program

                          • close write ends of both pipes
                          • enter a loop: read() from stdoutPipe.read
                            and accumulate into a buffer
                          • read() from stderrPipe.read similarly
                          • when both pipes hit EOF, exit loop
                          • waitpid(child PID, &status) to get exit code
                          • package { stdout, stderr, exitCode } as JSON
                          • send HTTP response back to the browser
```

### Why this pattern?

* **Pipes** let the kernel shuttle byte streams from the child’s stdout/stderr back to your daemon.
* **fork() + execvp()** is the classic Unix way to launch an external program in a new process.
* **dup2()** re-wires the child’s standard file descriptors into the pipes, so anything the child writes to `printf()`, `puts()`, or file-descriptor 1/2 goes straight back to you.
* **waitpid()** blocks until the child exits, giving you its return code.

### Step-by-step in pseudo-C:

1. `int outfd[2], errfd[2]; pipe(outfd); pipe(errfd);`
2. `pid = fork();`
3. **Child** (`pid == 0`):
   * `dup2(outfd[1], STDOUT_FILENO); dup2(errfd[1], STDERR_FILENO);`
   * Close all four pipe fds.
   * `execvp(cmd, argv);`
4. **Parent** (`pid > 0`):
   * Close the two “write” ends: `close(outfd[1]); close(errfd[1]);`
   * Read from `outfd[0]` and `errfd[0]` until EOF into two strings.
   * `waitpid(pid, &status, 0);` extract `WEXITSTATUS(status)`.
   * Build your JSON `{ "stdout": ..., "stderr": ..., "exitCode": ... }` and return it.

With this pattern embedded in your HTTP-request handler, *any* user-space command (ls, netstat, ping, etc.) will run under your daemon, and you’ll capture every byte of its output and its exit status—and can then sling it right back to your HTML/JS front-end.

---

### Q: How is the data read/parsed by the daemon and sent back to html ?

A call to `execvp(cmd, argv)` isn’t “running in a background shell” so much as it is “completely transforming the current process into the new program.” Here’s what actually happens when you do:

```c
pid_t pid = fork();
if (pid == 0) {
    // Child
    execvp(cmd, argv);
    // (If execvp succeeds, you never return here—this process *becomes* cmd.)
} else {
    // Parent
    // ... waitpid() or continue on its own thread
}
```

---

### How it differs from a “background shell”

1. **No shell involved**

   * You’re not invoking `/bin/sh -c "ping …"` (which would spawn a shell interpreter that then runs your command).
   * You’re directly loading the binary (`/usr/bin/ping`) into the child process. This avoids any shell parsing, expansions, or injection risks.

2. **Not a “job” or shell job**

   * In an interactive shell, if you type `ping &`, the shell forks and places the child in the background job table.
   * Here, you fork in your daemon and immediately `execvp`, but you don’t have any shell job control (`&`, job IDs, `%1`, SIGTSTP\`, etc.). It’s just: “spawn this process, get its PID, and either wait for it or collect its status later.”

3. **Daemon-managed lifecycle**

   * Your daemon is in charge of lifecycle: it decides whether to block on `waitpid()` (synchronous) or to track the child asynchronously and notify your HTML client later.
   * There’s no terminal or TTY attached, so no stdin, no job-control signals (SIGINT, SIGTSTP) will be delivered in the way an interactive shell would.

---

### Simplified Flow

```
[Daemon Parent]
   │
   ├─ create pipes for stdout/stderr
   ├─ fork()
   │
   ├──> [Child]
   │       │
   │       ├─ dup2 pipes onto fd 1/2
   │       └─ execvp(cmd, argv)
   │           (this process *becomes* the new program)
   │
   └──> [Back in Parent]
           │
           ├─ close write-ends of pipes
           ├─ read child’s output from pipes
           └─ waitpid(child) to get exit code
```

* The **child** never invokes a shell; it simply transforms itself into the exact binary you asked for.
* The **daemon** (parent) remains running the HTTP server code, keeping control of how and when you harvest the child’s stdout/stderr and report back to your web UI.

In short, `execvp` is how UNIX programs replace a process’s memory image with a new executable—not a background shell or job-control mechanism.

You don’t actually need to hard-code or “list out” every single binary on the system. Instead, you have two common patterns:

#### Rely on the Shell’s `PATH` Lookup via `execvp`

```c
char *argv[] = { "ls", "-l", NULL };
execvp(argv[0], argv);
```
* **What it does**:
  * The `execvp()` call looks at the child’s `PATH` environment variable (e.g. `/usr/local/bin:/usr/bin:/bin`) and tries each directory in turn until it finds an executable file named `ls`.
  * You don’t need to know its full path yourself (e.g. `/bin/ls`); the kernel will locate it for you.
* **Pros**:
  * Very simple.
  * Any command in the user’s `PATH` works (as long as it’s allow-listed).
* **Cons**:
  * If the user’s `PATH` is weird or attacker-controlled, you might inadvertently run the wrong binary.

---
