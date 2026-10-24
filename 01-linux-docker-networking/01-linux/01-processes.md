# Processes

> Phase 1: Linux, Docker & Networking › Linux

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

A **process** is an instance of a running program. When Linux executes a program, the kernel creates a process with its own execution state, memory space, security credentials, and allocated resources.

Processes are fundamental to Linux system administration and Kubernetes because containers ultimately run as isolated processes on the host operating system.

A process typically starts when an existing process requests the creation of a new process. Linux commonly uses `fork()` to create a child process and `exec()` to replace a process's program with another executable.

For example, when you run `python app.py`, your shell starts a process that executes the Python interpreter, which loads and runs your application.

Processes continue running until they finish, encounter an error, or receive a signal that causes them to terminate. They may also be stopped or paused temporarily.

## Key points

- **Process ID (PID):** A unique numeric identifier for a process within its PID namespace. Linux assigns PIDs when processes are created.
- **Parent Process ID (PPID):** Identifies the parent process that created the current process. Inspect it using `ps` or `/proc`.
- **Process lifecycle:** A process moves through states such as running, sleeping, stopped, and zombie. A terminated process may remain briefly as a zombie until its parent collects its exit status.
- **Process creation:** Linux uses system calls such as `fork()`, `clone()`, and `execve()` to create processes or execution contexts and run programs.
- **Foreground and background processes:** Foreground processes interact with the current terminal session. Background processes allow the shell to accept additional commands while they run.
- **Signals:** Signals communicate events or request actions. `SIGTERM` requests graceful termination, while `SIGKILL` forcibly terminates a process and cannot be caught or ignored.
- **Graceful shutdown:** Applications should handle termination signals, finish essential work, release resources, and exit cleanly when possible.
- **Process monitoring:** Tools such as `ps`, `top`, and `htop` show process information, resource consumption, and execution states.
- **Process scheduling:** The Linux kernel schedules runnable tasks to share CPU time. A process can exist without currently executing on a CPU.
- **Memory isolation:** Processes normally have separate virtual address spaces. Threads within a process share much of the same memory and resources.
- **Exit status:** A process returns an exit code when it terminates. Conventionally, `0` means success and a nonzero value indicates an error.
- **Orphan processes:** A process whose parent terminates is generally reparented to an appropriate system process or subreaper.
- **Zombie processes:** A process that has exited but whose termination status has not yet been collected by its parent. Zombies do not execute and typically consume little memory.
- **PID namespaces:** Containers can have their own PID namespace, allowing a process to see a different process-ID hierarchy from the host.
- **Container entrypoints:** A container's main process is particularly important. If its primary process exits, the container runtime generally considers the container stopped.
- **PID 1 behavior:** The first process in a PID namespace has special responsibilities, including adopting orphaned children and handling signals appropriately. A container's PID 1 should be designed to handle termination signals correctly.
- **Signal forwarding:** A container entrypoint that launches an application through a shell may fail to forward signals correctly unless it uses proper signal handling or `exec`.
- **Resource limits:** Linux mechanisms such as cgroups can constrain CPU and memory consumption. A process can be terminated by the kernel's out-of-memory handling if memory pressure triggers an OOM kill.

## Commands / manifests

### 1. Inspect running processes

```bash
# Display processes associated with the current terminal
ps

# Display all processes with detailed information
ps aux

# Show process hierarchy
ps -ef --forest

# Inspect a particular process by PID
ps -p 1234 -o pid,ppid,stat,etime,%cpu,%mem,cmd

# Display live process activity
top

# Find processes by name or command
pgrep -a python
```

Replace `1234` with the actual PID of the process you want to inspect.

### 2. Start and manage processes

```bash
# Start a process in the foreground
sleep 300

# Start a process in the background
sleep 300 &

# Display background jobs in the current shell
jobs -l

# Bring a background job to the foreground
fg %1

# Suspend a foreground process using the keyboard
# Press Ctrl+Z while the process is running

# Resume a suspended job in the background
bg %1

# Run a command independently of normal shell hangup handling
nohup sleep 300 > sleep.log 2>&1 &
```

`nohup` protects against the usual hangup signal, but it does not guarantee that a process survives every session, service-manager, or container shutdown.

### 3. Terminate processes

```bash
# Request graceful termination
kill -TERM 1234

# Equivalent shorthand for the default termination signal
kill 1234

# Force termination when graceful shutdown fails
kill -KILL 1234

# Terminate processes matching a command name
pkill -TERM -f "sleep 300"

# Check whether a process still exists
ps -p 1234
```

Always verify the PID before sending a signal. Prefer `SIGTERM` first and use `SIGKILL` only when necessary. A process in an uninterruptible kernel sleep may not disappear immediately, even after receiving `SIGKILL`.

### 4. Inspect process details through `/proc`

```bash
# View a process's status, parent PID and memory information
cat /proc/1234/status

# View the command line used to start a process
tr '\0' ' ' < /proc/1234/cmdline
echo

# Inspect the executable associated with a process
readlink /proc/1234/exe

# Display the current process's PID
echo $$

# Display the current shell's parent PID
echo $PPID
```

The `/proc` filesystem exposes kernel-maintained process and system information. Access to some details depends on permissions and system security settings.

### 5. Practice process lifecycle and exit codes

```bash
# Create a long-running process
sleep 300 &

# Save its PID
PID=$!

# Inspect its parent and state
ps -p "$PID" -o pid,ppid,stat,cmd

# Stop it gracefully
kill -TERM "$PID"

# Wait for it and collect its exit status
wait "$PID"
echo "Exit status: $?"
```

The exit status is available after `wait` collects the process's result. A successful `SIGTERM`-initiated termination is not necessarily reported as exit code `0`; shell behavior commonly represents signal termination as a value such as `143` for `SIGTERM`.

### 6. Investigate processes inside Docker

```bash
# Run a container with a long-running foreground process
docker run -d --name process-lab alpine sleep 300

# Display processes inside the container
docker top process-lab

# Open a shell and inspect processes in the container
docker exec -it process-lab sh

# Inspect the container's main process and configuration
docker inspect process-lab

# Send a graceful stop signal and allow time for shutdown
docker stop -t 10 process-lab

# Remove the stopped container
docker rm process-lab
```

Docker normally sends `SIGTERM` to the container's main process when stopping a container, then sends `SIGKILL` if the process does not exit within the configured grace period.

### 7. Understand the container's main process

```dockerfile
FROM python:3.12-slim

WORKDIR /app
COPY app.py .

# Run Python as the container's main process
CMD ["python", "app.py"]
```

The JSON-array form is called **exec form**. It launches the executable directly rather than wrapping the command in a shell. This generally makes signal delivery and shutdown behavior easier to reason about.

For a shell entrypoint, use `exec` when replacing the shell with the application:

```bash
#!/bin/sh
set -e

# Perform initialization here

exec python /app/app.py
```

The `exec` command replaces the shell process with Python, allowing Python to become the main process and receive termination signals directly.

## Common failures & fixes

- **Process is still running after closing the terminal:** Start long-running workloads with an appropriate service manager, terminal multiplexer, or background process strategy. For production applications, prefer `systemd` or a container orchestrator over ad hoc background commands.
- **`kill` reports "No such process":** The PID may have exited or changed. Recheck using `ps` or `pgrep` before retrying.
- **`kill` reports "Operation not permitted":** The current user lacks permission to signal the target process. Verify ownership and use authorized elevated privileges only when necessary.
- **Process ignores `SIGTERM`:** The application may not handle the signal correctly or may be blocked during shutdown. Inspect its logs and shutdown implementation. Use `SIGKILL` only as a last resort.
- **Zombie processes appear in `ps`:** Identify the parent process. The parent must collect the child's exit status using `wait()` or an equivalent mechanism. Killing the zombie itself does not solve the problem.
- **Container stops immediately after starting:** The container's main process may have completed, crashed, or failed during initialization. Inspect `docker logs`, `docker inspect`, and the exit code.
- **Container does not stop gracefully:** The application may not handle `SIGTERM`, or an entrypoint shell may interfere with signal delivery. Prefer exec-form commands and ensure the application supports graceful shutdown.
- **Container runs but the application is missing:** A process listing inside a container shows only the processes visible within its PID namespace. Compare `docker top` with the output of `ps` inside the container.
- **Application is terminated unexpectedly under memory pressure:** Inspect kernel logs and container memory limits. An OOM kill may terminate a process without allowing application-level cleanup.
- **Background job terminates when the shell exits:** Job-control processes can receive `SIGHUP` when their terminal or shell session ends. Use a service manager or `tmux`/`screen` for persistent interactive work.
- **High CPU consumption:** Identify the process using `top` or `ps`, inspect its command and logs, and determine whether it is performing expected work or stuck in a loop.
- **Process appears stuck in `D` state:** This usually indicates uninterruptible sleep, often associated with I/O. Investigate the underlying storage, filesystem, or kernel operation rather than repeatedly sending kill signals.
- **PID file points to the wrong process:** PIDs can be reused after processes exit. Validate the process identity rather than trusting a stale PID file alone.

## References

- [Linux `ps` manual](https://man7.org/linux/man-pages/man1/ps.1.html)
- [Linux `kill` manual](https://man7.org/linux/man-pages/man1/kill.1.html)
- [Linux `proc` filesystem documentation](https://docs.kernel.org/filesystems/proc.html)
- [Linux process creation: `fork(2)`](https://man7.org/linux/man-pages/man2/fork.2.html)
- [Linux program execution: `execve(2)`](https://man7.org/linux/man-pages/man2/execve.2.html)
- [Linux signal overview](https://man7.org/linux/man-pages/man7/signal.7.html)
- [Docker: Running containers](https://docs.docker.com/engine/containers/run/)
- [Docker: Container lifecycle and stop behavior](https://docs.docker.com/reference/cli/docker/container/stop/)
- [Kubernetes: Container lifecycle hooks](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/)
