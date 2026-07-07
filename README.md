# unix-shell

A POSIX-style Unix shell written from scratch in C. It implements its own tokenizer and CFG-based grammar checker, supports pipelines and I/O redirection, background execution with job control, and a handful of custom built-in commands — all built directly on top of `fork`, `exec`, `pipe`, `dup2`, and POSIX signal handling.

## Features

- **Custom parser** — a hand-written tokenizer plus a recursive-descent grammar checker validates commands (pipelines, redirections, `;`, `&&`, `&`) before anything is executed.
- **Pipelines** — chain any number of commands with `|`.
- **I/O redirection** — `<` for input, `>` to overwrite output, `>>` to append.
- **Sequential & conditional execution** — separate commands with `;`, or chain them with `&&`.
- **Background jobs** — append `&` to run a command (or pipeline) in the background.
- **Job control** — `activities`, `fg`, `bg`, and signal-based suspension (`Ctrl+Z`) / interruption (`Ctrl+C`) that only affect the foreground process, never the shell itself.
- **Command history** — every command is logged to disk and can be replayed.
- **Custom navigation & listing built-ins** — `hop` (a `cd` replacement with `-`/`~`/`..` support) and `reveal` (an `ls` replacement with `-a`/`-l` flags).
- **Signal-safe process groups** — every command/pipeline is placed in its own process group so signals and job control behave correctly.

## Build

```bash
make        # builds ./shell.out
make clean  # removes object files and the binary
```

Requires `gcc` and a POSIX-compliant environment (Linux/macOS). Built with `-std=c99 -Wall -Wextra -Werror`.

## Run

```bash
./shell.out
```

You'll be dropped into a prompt of the form:

```
<username@hostname:~/current/path> 
```

Type `exit` to quit.

## Built-in Commands

| Command | Description |
|---|---|
| `hop [dir]` | Change directory. Supports `~` (home), `-` (previous directory), `..` (parent), `.` (no-op), and chaining multiple arguments in one call. |
| `reveal [-a] [-l] [path]` | List directory contents, sorted lexicographically. `-a` shows hidden entries, `-l` prints one entry per line. Defaults to the current directory. |
| `log` | Print command history (up to the last 15 commands). |
| `log purge` | Clear command history. |
| `log execute <n>` | Re-run the *n*th most recent command from history. |
| `activities` | List all background/stopped jobs with their PID and state. |
| `ping <pid> <signal>` | Send a signal number to a process by PID. |
| `fg [job]` | Bring a background/stopped job to the foreground (defaults to the most recent). |
| `bg [job]` | Resume a stopped job in the background. |
| `exit` | Exit the shell, killing any remaining background jobs. |

Any command not listed above is treated as an external program and executed via `execvp`.

## Operators

| Syntax | Meaning |
|---|---|
| `cmd1 \| cmd2` | Pipe the output of `cmd1` into `cmd2` |
| `cmd < file` | Redirect input from `file` |
| `cmd > file` | Redirect output to `file` (overwrite) |
| `cmd >> file` | Redirect output to `file` (append) |
| `cmd &` | Run `cmd` in the background |
| `cmd1 ; cmd2` | Run `cmd1`, then `cmd2`, regardless of `cmd1`'s exit status |
| `cmd1 && cmd2` | Run `cmd2` only after `cmd1` |

These can be combined, e.g.:

```bash
sort < input.txt | uniq >> output.txt &
grep foo file.txt ; echo done
```

## Job Control

- **Ctrl+C** (`SIGINT`) interrupts only the current foreground process — the shell keeps running.
- **Ctrl+Z** (`SIGTSTP`) stops the foreground process and moves it into the background job list.
- Use `activities` to see running/stopped jobs, `fg` to resume one in the foreground, and `bg` to resume a stopped job in the background.

## Project Structure

```
.
├── Makefile
├── src/
│   ├── main.c        # prompt loop, tokenizer, grammar checker, top-level dispatch
│   ├── commands.c     # pipeline/redirection parsing into Command structs
│   ├── execute.c      # forks and wires up pipes/redirection, runs pipelines
│   ├── hop.c           # cd-like directory navigation
│   ├── reveal.c        # ls-like directory listing
│   ├── activities.c    # job table, signal handlers, ping/fg/bg
│   └── log.c            # persistent command history
└── include/            # corresponding headers
```

## Notes

- Command history is persisted to `history.log` in the working directory.
- Background job status changes (finished/exited) are reported the next time the prompt is redrawn.
