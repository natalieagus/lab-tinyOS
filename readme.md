# tinyOS for the Beta Processor

A small, single address space timesharing kernel for the Beta CPU (UASM, runs in BSIM). Based on the 6.004 OS demo (Steve Ward, Fall 1994), extended with mouse support and non-blocking I/O checks.

## What it does

Boots into a round-robin scheduler running four user processes that share the kernel address space, each with its own register state (in `ProcTbl`) and its own stack:

| Proc | Role                                                                                              |
| ---- | ------------------------------------------------------------------------------------------------- |
| P0   | Prompts, reads a line from the keyboard, upper-cases it, pipes characters to P1                   |
| P1   | Reads from P0 through a bounded-buffer FIFO, translates words to Pig Latin, writes to the console |
| P2   | Increments `Count3` on each quantum and yields; the count is printed as the P0 prompt             |
| P3   | Waits on a mouse semaphore, calls `GetMouse`, prints click coordinates in hex                     |

P0 and P1 are connected by a 100-slot FIFO synchronised with two semaphores (`Chars`, `Holes`). A `Prompt` semaphore prevents P0 from re-prompting until P1 has finished translating the previous line.

## Memory and state

- `UserMState`: 32-word region holding the saved registers of the interrupted process while the kernel runs.
- `ProcTbl`: 31-word slot per process (R0 through R28, SP, XP/PC). R31 is omitted since it is always zero.
- `CurProc`: pointer into `ProcTbl` for the currently swapped-in process.
- `KStack`: dedicated kernel stack, installed on every kernel entry.

## Interrupt vectors

| Vector      | Handler   | Notes                                                                                      |
| ----------- | --------- | ------------------------------------------------------------------------------------------ |
| `VEC_RESET` | `I_Reset` | Starts P0                                                                                  |
| `VEC_II`    | `I_IllOp` | Illegal instruction, includes SVC dispatch                                                 |
| `VEC_CLK`   | `I_Clk`   | Lightweight: only saves R0 unless `TicsLeft` goes negative, then full save and `Scheduler` |
| `VEC_KBD`   | `I_Kbd`   | Lightweight: stores keycode in `Key_State`                                                 |
| `VEC_MOUSE` | `I_Mouse` | Lightweight: stores packed x/y click in `Mouse_State`                                      |

`ENTER_INTERRUPT` adjusts XP by -4 to account for the pipeline bubble before the handler runs.

The clock handler gives each process `Tics` (= 2) clock interrupts per quantum. The keyboard and mouse handlers only touch R0, so they skip the full state save and always return to the interrupted process.

## SVC table

User code traps into the kernel via reserved opcodes. The bottom 4 bits of the faulting instruction index `SVCTbl`:

| SVC | Macro             | Behaviour                                                                        |
| --- | ----------------- | -------------------------------------------------------------------------------- |
| 0   | `Halt()`          | Blocks the calling process (re-executes on resume)                               |
| 1   | `WrMsg()`         | Print zero-terminated text immediately following the SVC, advance XP past it     |
| 2   | `WrCh()`          | Print character in R0                                                            |
| 3   | `GetKey()`        | Blocking read from `Key_State`, clears buffer on return                          |
| 4   | `HexPrt()`        | Print R0 as 8 hex digits                                                         |
| 5   | `Wait(S)`         | P operation on semaphore at address in R3, blocks on zero                        |
| 6   | `Signal(S)`       | V operation on semaphore at address in R3                                        |
| 7   | `Yield()`         | Reschedule immediately                                                           |
| 8   | `GetMouse()`      | Blocking read from `Mouse_State`, returns packed (x<<16)\|y, resets buffer to -1 |
| 9   | `CheckMouse()`    | Non-blocking read from `Mouse_State`, does not clear                             |
| 10  | `CheckKeyboard()` | Non-blocking read from `Key_State`, does not clear                               |

Blocking SVCs return through `I_Wait`, which rewinds XP by 4 so the SVC re-executes when the process is next scheduled.

## Scheduler

`Scheduler` copies `UserMState` into the current `ProcTbl` slot, advances `CurProc` (wrapping at the end of the table), copies the new slot back into `UserMState`, resets `TicsLeft`, and returns. The clock handler invokes it on quantum exhaustion; `Yield` invokes it on demand.

## Synchronisation pattern in P3

P3 blocks on `MouseSemaphore`, which is only signalled by P0 after it has confirmed an actual click via `CheckMouse`. P0 busy-waits between `CheckMouse` and `CheckKeyboard` so that:

1. A pending mouse click is only consumed once (P0 signals `MouseSemaphore` exactly when there is fresh mouse data).
2. Keyboard input still drives normal line entry.
3. P3 cannot fire on stale `Mouse_State` data after a prior click, because P0 controls the signalling and the kernel resets `Mouse_State` to -1 inside `GetMouse`.

P3 signals `Prompt` itself after printing, so the prompt loop resumes whether the user clicked or typed.

## Build and run

This is BSIM input. The file uses:

```
.include beta.uasm
.options clock tty
```

Load `tinyOS.uasm` into BSIM, enable the keyboard and mouse devices, and run. Expected console behaviour:

```
Start typing, Bunky.

00000000> hello
ELLOHAY
```

Mouse clicks produce lines such as:

```
Click at x = 0000004A, y = 0000001F
```

The leading hex value on the prompt is `Count3`, the number of quanta P2 has consumed.

## File layout

- Kernel: interrupt vectors, `UserMState`, `SAVESTATE`/`RESTORESTATE`, `I_IllOp` and `UUOTbl`/`SVCTbl`, scheduler, utility routines (`KHexPrt`, `KWrMsg`, `KMsgAux`).
- User mode helpers: `Wait`, `Signal`, `semaphore`, `UCase`, `VowelP`, `Send`, `Rcv`.
- User processes: P0, P1, P2, P3, each with its own `STORAGE(256)` stack.

## Extensions over the original demo

- Mouse IRQ handler (`I_Mouse`) and `Mouse_State` buffer.
- `GetMouse`, `CheckMouse`, `CheckKeyboard` SVCs.
- Fourth `ProcTbl` entry and P3 for mouse click display.
- `MouseSemaphore` plus P0 polling loop so P3 only fires on genuinely new clicks and does not double-print after a typed line.
