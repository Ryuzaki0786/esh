# esh — Unix Shell

A fully functional Unix shell implemented in C using only POSIX system calls.

## Features
- Process creation via fork/exec/wait
- Multi-stage pipes between processes
- I/O redirection with dup2
- Signal handling (Ctrl+C, Ctrl+Z)

## Build

make
./esh

## What this demonstrates
Built without external libraries or frameworks. Every feature touches the kernel directly, covering the hardware-software boundary from process model to file descriptor management.
