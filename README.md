# esh — Unix Shell

A Unix shell written in C using only POSIX system calls — no external libraries or frameworks.

## Build & Run

```c
gcc esh.c -o esh
./esh
```

## Features

- **Command execution** — tokenizes input and calls `fork()`/`execvp()`/`waitpid()`
- **Pipes** (`ls | grep .c`) — `pipe()` + `dup2()` rewires stdout of left command to stdin of right command
- **Output redirection** (`ls > out.txt`) — opens file with `O_WRONLY | O_CREAT | O_TRUNC`, `dup2()` to stdout
- **Input redirection** (`sort < input.txt`) — opens file with `O_RDONLY`, `dup2()` to stdin
- **Signal handling** — parent ignores `SIGINT` (Ctrl+C); child restores default so commands can be interrupted
- **Built-in commands** — `cd` (with `chdir()`), `exit`

## How it works

The main loop reads input with `fgets()`, detects operators (`|`, `>`, `<`) via `strchr()`, and dispatches accordingly. For external commands, the parent forks a child — the child calls `execvp()` which replaces the process image entirely, inheriting the rewired file descriptors.

The key insight behind pipes:
ls (child 1)               grep (child 2)

↓                           ↑

stdout                      stdin

↓                           ↑

fd[1] ──→ [ pipe buffer ] ──→ fd[0]
`dup2(fd[1], 1)` makes stdout point to the pipe write end. `dup2(fd[0], 0)` makes stdin point to the pipe read end. Both ends are then closed in both children — leaving only the dup'd descriptors open.

## Common interview questions this project covers

**Why do you close both pipe ends in the child after dup2?**
After `dup2(fd[1], 1)`, file descriptor 1 already points to the pipe. The original `fd[1]` is now a duplicate — if you leave it open, the read end of the pipe will never see EOF because there's still an open write end. Closing unused ends is mandatory for correct pipe behavior.

**What happens if you don't call waitpid()?**
The child becomes a zombie process — it finishes execution but its exit status stays in the process table because the parent never collected it. Over time this leaks process table entries.

**Why does the parent ignore SIGINT but the child restores it?**
If the parent didn't ignore SIGINT, pressing Ctrl+C while running a command would kill the shell itself. The child restores the default handler so the running command (e.g. `cat`) can be interrupted normally, without affecting the shell.

**What is execvp doing exactly?**
It replaces the current process image with a new program — the child process stops being a copy of esh and becomes `ls` or whatever command was typed. The file descriptors are inherited across exec, which is why dup2 works.

**Why fork() before exec()?**
exec() is destructive — it replaces the current process. If the shell called exec() directly, the shell would be gone. fork() creates a disposable copy; the copy calls exec() and gets replaced, while the parent (the shell) waits and continues.
