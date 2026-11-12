# 01.asm — Basic PUSH/POP Stack Behavior
## Stack Operations Trace

| Instruction      | RSP Before | RSP After  | Effect on memory                       |
|------------------|------------|------------|----------------------------------------|
| `mov rax, 100`   | RSP₀       | RSP₀       | RAX = 100                              |
| `mov rbx, 200`   | RSP₀       | RSP₀       | RBX = 200                              |
| `push rax`       | RSP₀       | RSP₀ − 8   | [RSP₀ − 8] = 100  (lowest address so far) |
| `push rbx`       | RSP₀ − 8   | RSP₀ − 16  | [RSP₀ − 16] = 200 (new top of stack)   |
| `nop`            | RSP₀ − 16  | RSP₀ − 16  | (no change)                            |
| `pop rcx`        | RSP₀ − 16  | RSP₀ − 8   | RCX = 200  (last pushed = first popped)|
| `pop rdx`        | RSP₀ − 8   | RSP₀       | RDX = 100                              |
| `nop`            | RSP₀       | RSP₀       | (no change)                            |
| `mov rax, 60`    | RSP₀       | RSP₀       | exit syscall number                    |
| `xor rdi, rdi`   | RSP₀       | RSP₀       | RDI = 0 (exit status)                  |
| `syscall`        | RSP₀       | —          | process exits                          |

## How the Stack Is Used
- **PUSH grows the stack downward.** Each `push` subtracts 8 from RSP (64-bit
  operand) and stores the value at the new, lower address.
- The *first* push (RAX = 100) lands at the higher address RSP₀ − 8.
- The *second* push (RBX = 200) lands at RSP₀ − 16, which is now the top.
- **POP reads from the top and grows the stack upward** by 8 each time.
  Because the last value pushed (200) is at the top, it comes out first:
  `pop rcx` → 200, then `pop rdx` → 100.

## Key Observations
- Net stack movement: −16 then +16 → **RSP returns exactly to RSP₀**.
  The stack is perfectly balanced.
- Final register values: RAX=60, RBX=200, RCX=200, RDX=100.
  Note RAX is overwritten at the end with 60, so the original 100
  survives only in RDX after the pops.
- LIFO order is the core lesson: pushed order was 100, 200; popped order
  was 200, 100.
- No `call`/`ret` is used, so the only stack traffic is the four explicit
  push/pop instructions.

# 02.asm — Call/Ret and a Standard Stack Frame
## Initial Stack State
- At `_start`, RSP = RSP₀ (address set by the kernel; ASLR-dependent).
- RBP is not initialized/meaningful before `work` sets it up.

## Stack Operations Trace (call site → callee → return)

| Instruction            | RSP            | Effect on stack / memory                       |
|------------------------|----------------|------------------------------------------------|
| `call work`            | RSP₀ → RSP₀−8  | Pushes return address (address of `mov rax,60`) |
| `push rbp`             | RSP₀−8 → RSP₀−16 | Saves caller's RBP at [RSP₀−16]             |
| `mov rbp, rsp`         | RSP₀−16        | RBP = RSP₀−16 (new frame base)                 |
| `sub rsp, 16`          | RSP₀−16 → RSP₀−32 | Reserves 16 bytes for locals                |
| `mov [rbp−8], 100`     | RSP₀−32        | [RSP₀−24] = 100  (first local)                 |
| `mov [rbp−16], 200`    | RSP₀−32        | [RSP₀−32] = 200  (second local)                |
| `mov rax, [rbp−8]`     | RSP₀−32        | RAX = 100                                      |
| `mov rbx, [rbp−16]`    | RSP₀−32        | RBX = 200                                      |
| `nop`                  | RSP₀−32        | —                                              |
| `mov rsp, rbp`         | RSP₀−32 → RSP₀−16 | Discards locals (RSP back to frame base)   |
| `pop rbp`              | RSP₀−16 → RSP₀−8  | RBP restored from [RSP₀−16]                |
| `ret`                  | RSP₀−8 → RSP₀  | Pops return address into RIP, jumps back       |
| `mov rax, 60`          | RSP₀           | —                                              |
| `xor rdi, rdi`         | RSP₀           | RDI = 0                                        |
| `syscall`              | RSP₀           | process exits                                  |

# 03.asm — Minimal CALL/RET Without a Frame

## Initial Stack State
- At `_start`, RSP = RSP₀ (kernel-provided stack top).
- RBP is never touched in this program.

## Stack Operations Trace

| Instruction        | RSP          | Effect on stack / registers                  |
|--------------------|--------------|----------------------------------------------|
| `mov rax, 10`      | RSP₀         | RAX = 10                                     |
| `call calculate`   | RSP₀ → RSP₀−8 | Pushes return address (= address of `after_call`) |
| `add rax, 5`       | RSP₀−8       | RAX = 15 (operates on the *caller's* RAX)    |
| `ret`              | RSP₀−8 → RSP₀ | Pops return address into RIP, jumps to `after_call` |
| `nop`              | RSP₀         | —                                            |
| `mov rax, 60`      | RSP₀         | RAX = 60 (exit syscall number)               |
| `xor rdi, rdi`     | RSP₀         | RDI = 0                                      |
| `syscall`          | RSP₀         | process exits                                |



## Key Observations
- Stack delta over the whole call: **−8 (call) then +8 (ret) = 0.**
  RSP is perfectly balanced.
- Exactly one 8-byte slot is ever used on the stack, and it holds only
  the return address.
- There is no way for the callee to address its own "frame" — it doesn't
  need one. This is the contrast to `02.asm`, where `sub rsp, 16`
  reserved memory.
- After `call` returns, RAX = 15 as the comment predicts. (RAX is then
  overwritten with 60 for exit; in GDB, break at `after_call` to see 15.)
- If you inspect `x/1gx $rsp` at the very first instruction of
  `calculate`, you'll see the return address — i.e. the address of the
  `nop` after `after_call`.

# 04.asm — 32-bit Standard Frame (ESP/EBP)

## Initial Stack State
- `BITS 32`; Linux i386. At `_start`, ESP = ESP₀ (kernel-set).
- EBP is not meaningful before `work` sets it up.
- Operand size is 4 bytes here, so each push/sub moves ESP by 4.

## Stack Operations Trace

| Instruction           | ESP             | Effect on stack / memory                     |
|-----------------------|-----------------|----------------------------------------------|
| `call work`           | ESP₀ → ESP₀−4   | Pushes return address                        |
| `push ebp`            | ESP₀−4 → ESP₀−8 | Saves caller's EBP at [ESP₀−8]               |
| `mov ebp, esp`        | ESP₀−8          | EBP = ESP₀−8 (frame base)                    |
| `sub esp, 8`          | ESP₀−8 → ESP₀−16| Reserves 8 bytes for two locals              |
| `mov [ebp−4], 100`    | ESP₀−16         | [ESP₀−12] = 100  (local #1)                  |
| `mov [ebp−8], 200`    | ESP₀−16         | [ESP₀−16] = 200  (local #2)                  |
| `mov eax, [ebp−4]`    | ESP₀−16         | EAX = 100                                    |
| `mov ebx, [ebp−8]`    | ESP₀−16         | EBX = 200                                    |
| `nop`                 | ESP₀−16         | —                                            |
| `mov esp, ebp`        | ESP₀−16 → ESP₀−8| Discards locals                              |
| `pop ebp`             | ESP₀−8 → ESP₀−4 | EBP restored from [ESP₀−8]                   |
| `ret`                 | ESP₀−4 → ESP₀   | Pops return address into EIP                 |
| `mov eax, 1`          | ESP₀            | exit syscall number                          |
| `xor ebx, ebx`        | ESP₀            | EBX = 0 (exit status)                        |
| `int 0x80`            | ESP₀            | process exits                                |


# 05.asm — 64-bit Push/Pop with Wide and Narrow Operands

## Initial Stack State
- At `_start`, RSP = RSP₀ (ASLR-dependent).
- RBP unused.

## Stack Operations Trace

| Instruction                    | RSP          | Effect on memory / registers                 |
|--------------------------------|--------------|----------------------------------------------|
| `mov rax, 0x1122334455667788`  | RSP₀         | RAX = full 64-bit constant                   |
| `mov rbx, 0x1234`              | RSP₀         | RBX = 0x1234 (upper 48 bits zero)            |
| `push rax`                     | RSP₀ → RSP₀−8 | [RSP₀−8] = 0x1122334455667788               |
| `push rbx`                     | RSP₀−8 → RSP₀−16 | [RSP₀−16] = 0x0000000000001234            |
| `nop`                          | RSP₀−16      | —                                            |
| `pop rbx`                      | RSP₀−16 → RSP₀−8 | RBX = 0x1234 (last pushed → first popped) |
| `pop rax`                      | RSP₀−8 → RSP₀ | RAX = 0x1122334455667788                    |
| `mov rax, 60`                  | RSP₀         | RAX = 60 (exit syscall)                      |
| `xor rdi, rdi`                 | RSP₀         | RDI = 0                                      |
| `syscall`                      | RSP₀         | process exits                                |


## Key Observations
- Net RSP movement: −16 then +16 → **RSP ends at RSP₀**, balanced.
- `x/2gx $rsp` just before the first pop shows exactly:
  `0x...: 0x0000000000001234  0x1122334455667788`
  (low address first) — this is a nice illustration of the "small"
  value still occupying a full 8-byte slot.
- After the pops, RAX and RBX are restored to their pre-push values
  (0x1122334455667788 and 0x1234). RAX is then overwritten with 60
  for exit.
- Contrast with `01.asm`: same structure, but the wide constant makes
  it visually obvious that a push consumes a full 8-byte slot no matter
  how small the source value is.

 
 
  # 06.asm — 32-bit Push/Pop with Wide and Narrow Operands

## Initial Stack State
- `BITS 32`; Linux i386. At `_start`, ESP = ESP₀.
- EBP unused.

## Stack Operations Trace

| Instruction            | ESP           | Effect on memory / registers               |
|------------------------|---------------|--------------------------------------------|
| `mov eax, 0x11223344`  | ESP₀          | EAX = 0x11223344                           |
| `mov ebx, 0x1234`      | ESP₀          | EBX = 0x00001234                           |
| `push eax`             | ESP₀ → ESP₀−4 | [ESP₀−4] = 0x11223344                      |
| `push ebx`             | ESP₀−4 → ESP₀−8 | [ESP₀−8] = 0x00001234                    |
| `nop`                  | ESP₀−8        | —                                          |
| `pop ebx`              | ESP₀−8 → ESP₀−4 | EBX = 0x1234 (last pushed → first popped)|
| `pop eax`              | ESP₀−4 → ESP₀ | EAX = 0x11223344                           |
| `mov eax, 1`           | ESP₀          | EAX = 1 (exit syscall number)              |
| `xor ebx, ebx`         | ESP₀          | EBX = 0                                    |
| `int 0x80`             | ESP₀          | process exits                              |

## How the Stack Is Used
- Each push is **4 bytes**, so ESP drops by 4 per push. The stack holds,
  top-down: 0x00001234 at [ESP₀−8] and 0x11223344 at [ESP₀−4].
- Pops are 4 bytes and in reverse order: 0x1234 first (into EBX), then
  0x11223344 (into EAX). As in `05.asm`, the pops restore the registers
  to the values they had before the pushes.
- The narrow 0x1234 still occupies a full 4-byte slot; its upper 16 bits
  are zero in the stored dword.

## Key Observations
- Net ESP movement: −8 then +8 → **ESP returns to ESP₀**, balanced.
- `x/2wx $esp` just before the first pop shows:
  `0x...: 0x00001234  0x11223344`
  (low address = last pushed first).
- Everything is **half the size of `05.asm`**: 4-byte pushes and a
  −8 total instead of −16.
- After the pops, EAX = 0x11223344 and EBX = 0x1234; EAX is then
  overwritten with 1 and EBX with 0 for the exit syscall.
- This pairs with `05.asm` to show the size difference between 32-bit
  and 64-bit stack traffic with identical source structure.