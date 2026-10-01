# MUL – Flags after unsigned multiplication

I ran `mul1.asm` and `mul2.asm` in GDB and checked `info registers eflags` straight after each `mul`.

## MUL works differently from ADD and SUB

When you multiply two numbers, the answer can be up to **twice as wide** as the inputs. So MUL saves the result in two halves:

| Size | Inputs | Result goes into |
|------|--------|------------------|
| 8-bit | AL × byte | **AH:AL** (that is, AX) |
| 16-bit | AX × word | **DX:AX** |
| 32-bit | EAX × dword | **EDX:EAX** |

Since the answer always has enough room, MUL can't really "overflow". So **CF and OF** answer one simple question instead:

> **Is the upper half of the answer used?**
> - Upper half is all zeros → CF = 0 and OF = 0 (the answer fits in the lower half alone)
> - Upper half is not zero → CF = 1 and OF = 1 (you need both halves)

CF and OF always match after MUL.

**SF, ZF, PF and AF are "undefined" after MUL.** That's the Intel manual's word for it. It means the CPU can leave anything in them and they don't tell you anything about the answer, so you shouldn't rely on them. You might see some of them set in GDB, and a different computer might show different values.

## How to build and run

```bash
nasm -f elf32 -g mul1.asm -o mul1.o
ld -m elf_i386 mul1.o -o mul1
gdb ./mul1
(gdb) break _start
(gdb) run
(gdb) stepi            # until the mul has run
(gdb) info registers eflags
(gdb) info registers eax edx
```

---

## Program 1: `mul1.asm` (8-bit)

```asm
mov al, [num1]     ; AL = 25
mul byte [num2]    ; AX = AL * 10
```

**Result:** 25 × 10 = 250 → AX = `0x00FA`, so **AH = 0x00** and **AL = 0xFA**

**GDB showed:** `eflags 0x286 [ PF SF IF ]`

| Flag | Status | Why |
|------|--------|-----|
| CF | **0 – cleared** | The upper half (AH) is 0. 250 fits in AL by itself, because a byte holds up to 255. |
| OF | **0 – cleared** | Same reason. OF always matches CF after MUL. |
| SF | set in GDB, but **undefined** | My CPU happened to set it because bit 7 of AL (`1111 1010`) is 1. That doesn't mean the answer is negative: MUL is unsigned and 250 is positive. Don't read anything into it. |
| PF | set in GDB, but **undefined** | AL has six 1s, which is even, so the CPU set it. Again, the manual says not to trust it after MUL. |
| ZF, AF | 0 in GDB, but **undefined** | Not meaningful after MUL. |

**Main lesson:** 250 is close to the 255 limit but stays under it, so AH is empty and CF = OF = 0. That tells your program it can just use AL and ignore AH.

---

## Program 2: `mul2.asm` (16-bit)

```asm
mov ax, [num1]     ; AX = 3000
mul word [num2]    ; DX:AX = AX * 200
```

**Result:** 3000 × 200 = 600,000 = `0x000927C0`
- **DX = 0x0009** (upper half)
- **AX = 0x27C0** (lower half)

**GDB showed:** `eflags 0xa07 [ CF PF IF OF ]`, plus `eax = 0x27c0`, `edx = 0x9`

| Flag | Status | Why |
|------|--------|-----|
| CF | **1 – set** | 600,000 is bigger than 65,535 (the most 16 bits can hold), so part of the answer spilled into DX. DX = 9, which isn't zero, so CF = 1. |
| OF | **1 – set** | Same reason. It always matches CF after MUL. |
| PF | set in GDB, but **undefined** | Not meaningful after MUL. |
| SF, ZF, AF | 0 in GDB, but **undefined** | Not meaningful after MUL. |

**Main lesson:** if you only looked at AX, you'd think the answer was `0x27C0` = 10,176, which is wrong. CF = 1 is the CPU's way of telling you "the real answer is bigger, go and look at DX too." That's why the program saves both AX **and** DX into `result`.

### Comparing the two programs

|  | mul1 | mul2 |
|--|------|------|
| Answer | 250 | 600,000 |
| Upper half | AH = 0 | DX = 9 |
| CF / OF | 0 / 0 | 1 / 1 |
| Need the upper half? | No | Yes |
