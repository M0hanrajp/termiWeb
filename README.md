# termiWeb :pager:

A lightweight Linux service that exposes a simple web‑based terminal UI. Users type arbitrary approved commands (e.g., `ping`, `ls`, `netstat`) into an HTML/JS front‑end; behind the scenes, a C daemon receives each command via an HTTP POST, securely spawns the requested binary, captures its stdout/stderr, and returns the results as JSON for display.

![Image](https://github.com/user-attachments/assets/60630348-aa67-429b-aa32-e097bea633d4)

## 2. Use Cases

1. **Learning & Prototyping**: Explore HTTP servers, JSON APIs, and Unix process management in C without exposing a full shell.
2. **Remote Diagnostics**: Provide a restricted, web‑accessible interface to run network or system‑info commands on a headless server.
3. **Embedded Systems**: Offer simple command invocation on devices (e.g., routers) with minimal UI.
4. **Custom Tooling**: Build atop this foundation to add features like streaming, authentication, or plugin hooks.

## 3. High‑Level Architecture

```
[Browser UI] ⇄ HTTP(S) ⇄ [C Daemon HTTP Server] ⇄ fork/exec ⇄ [Child Process] ⇄ pipes ⇄ stdout/stderr
```
* **Browser UI (HTML/JS)**: Input box + output pane; `fetch()` → `/run` endpoint.
* **C Daemon**:

  * Static file server (serves HTML/JS assets).
  * API endpoint (`POST /run`) parses JSON `{ cmd, args }`.
  * Validate against allow‑list.
  * `pipe()` + `fork()` + `execvp()` (or `execv()` with absolute path).
                               ▼  * Collect output and exit code; serialize to JSON.
* **Child Process**: Executes the approved binary; writes stdout/stderr to pipes.

a generic, user-space command you run via your HTML/JS “shell” — whether it’s `ls`, `netstat`, `curl`, or something else. 

```
+----------------+          +----------------------+          +---------------------+
|                | 1. Type  |                      | 2. JSON  |                     |
| Browser UI     | “ls -l”  | HTML/JS App (fetch)  | ───────> | Browser HTTP Client |
| (input & <pre>)|          |                      |          | + OS TCP stack      |
+----------------+          +----------------------+          +-----------┬---------+
                                                                           │
                                                                           │ HTTP POST over TCP
                                                                           │
                                                                           ▼
                                            +-----------------------------------------+
                                            |  C Daemon HTTP Server (libmicrohttpd)   |
                                            |  • Accepts POST /run                    |
                                            |  • Parses JSON { cmd:"ls", args:["-l"]} |
                                            +--------------------------┬--------------+
                                                                       │
                                                                       │ fork()/execvp()
                                                                       │
                              +-----------------------------------------------------------+
                              |  Kernel Process Management & Scheduling                   |
                              |  • fork() creates new PID                                 |
                              |  • execvp() loads /usr/bin/ls into that PID               |
                              |  • Kernel assigns CPU time, memory, permissions, etc.     |
                              +------------------------------┬----------------------------+
                                                             │
                                                             │ syscalls for any I/O:
                                                             │  • File API (open/read) via VFS
                                                             │  • Network sockets via net stack
                                                             │  • Other device interactions
                                                             │
                                                             ▼
                              +------------------------------+----------------------------+
                              |   Child Process (`ls `)                                   |
                              |   • Runs entirely in user-space                           |
                              |   • Does its own stdio: writes stdout/stderr to its fds   |
                              +--------------------┬--------------------------------------+
                                                   │
                           stdout/stderr pipes     │ read() by daemon
                                                   ▼
+--------------------------+-----------------------+--------------------------+
|  C Daemon                                                                   |
|  • Reads pipe buffers                                                       |
|  • Waits on child, collects exit code                                       |
|  • Serializes { stdout, stderr, exitCode } → JSON                           |
+--------------------------+-----------------------+--------------------------+
                                                   │
                                                   │ HTTP Response over TCP
                                                   ▼
+----------------+          +-----------------------+           +----------------+
|                | 7. JS    |                       | 8. Render |                |
| Browser HTTP   |◀─────────| HTML/JS App          ◀────────── | Browser UI     |
| Client + TCP   | Response | (parse JSON & update) |           | (<pre> output) |
+----------------+          +-----------------------+           +----------------+
```
## 4. Detailed Requirements

### 4.1 Functional Requirements

* Serve static HTML/CSS/JS from `localhost` (configurable port).
* Expose `POST /run` accepting JSON `{ cmd: string, args: string[] }`.
* Maintain an allow‑list of permitted commands.
* For each request:
  * Validate command.
  * Create stdout/stderr pipes.
  * Fork & redirect descriptors.
  * Exec the binary.
  * Read from pipes until EOF.
  * Wait for exit; gather exit code.
  * Respond with `{ stdout, stderr, exitCode }`.
* Handle errors (invalid JSON, disa allowed commands, timeouts) gracefully with HTTP error codes.

### 4.2 Non‑Functional Requirements

* **Security**: no shell interpolation; binaries run under restricted privileges or namespace.
* **Performance**: lightweight — minimal dependencies, low memory footprint.
* **Reliability**: clean resource cleanup; prevent zombie processes.
* **Extensibility**: easy to add commands to the allow‑list; placeholder for authentication or streaming.

## 5. Technology & Libraries

| Layer         | Language/Library                                                                  | Purpose                       |
| ------------- | --------------------------------------------------------------------------------- | ----------------------------- |
| Front‑end     | HTML5, CSS, JavaScript                                                            | UI, input box, output pane    |
|               | Fetch API                                                                         | HTTP requests                 |
|               | (Optional) WebSocket API                                                          | Real‑time streaming           |
| Static Server | Built‑in in C Daemon                                                              | Serve files                   |
| HTTP Server   | [libmicrohttpd](https://www.gnu.org/software/libmicrohttpd/) or Mongoose/CivetWeb | Lightweight HTTP server in C  |
| JSON          | [jansson](https://github.com/akheron/jansson) or cJSON                            | Parsing & serialization       |
| Process Mgmt. | POSIX APIs (`fork()`, `execvp()`, `pipe()`, `waitpid()`)                          | Launch and capture subprocess |
| Build Tools   | `gcc`/`clang`, `make`                                                             | Compilation                   |
| Deployment    | systemd unit                                                                      | Daemonize on startup          |

### 5.1 Security Considerations

* **Allow‑list only**: no arbitrary code execution.
* **Avoid shell**: use `execvp()` or `execv()` (no `/bin/sh -c`).
* **Privilege separation**: run daemon as unprivileged user; consider `chroot` or Linux namespaces for sandboxing.
* **CORS / Same‑Origin**: serve both UI and API from same origin, or set strict CORS headers.
* **Timeouts**: enforce command‑level time limits to avoid hanging processes.

---

_This is a wip readme, will be updated overtime!_

_for more questions on this refer here_

