# DIV – Flags after unsigned division

I ran `div1.asm` and `div2.asm` in GDB and checked `info registers eflags` before and after each `div`.

## DIV works differently from the others

DIV always takes a **double-width** number and divides it by your operand. Then it gives back two answers, a quotient and a remainder:

| Size | Dividend | Quotient goes into | Remainder goes into |
|------|----------|--------------------|---------------------|
| 8-bit | AX | AL | AH |
| 16-bit | DX:AX | AX | DX |
| 32-bit | EDX:EAX | EAX | EDX |

**What about the flags?** The Intel manual says **all six arithmetic flags (CF, OF, SF, ZF, AF, PF) are undefined after DIV.** That means DIV doesn't report on its result using the flags at all. Whatever you see in EFLAGS afterwards doesn't tell you anything about the division.

So why doesn't DIV need flags?
- It can't carry out or borrow like ADD and SUB, because the answer is always smaller than the number you started with.
- If something does go wrong, it doesn't set a flag. It **crashes the program**. If you divide by zero, or the quotient is too big for its register, the CPU raises a **divide error** (#DE) and Linux kills the program with "Floating point exception". There's no way for the error to slip by quietly, so there's no need for a flag to warn you.

## How to build and run

```bash
nasm -f elf32 -g div1.asm -o div1.o
ld -m elf_i386 div1.o -o div1
gdb ./div1
(gdb) break _start
(gdb) run
(gdb) stepi            # until the div has run
(gdb) info registers eflags
(gdb) info registers eax edx
```

---

## Program 1: `div1.asm` (8-bit)

```asm
mov ax, [dividend]   ; AX = 100
mov bl, [divisor]    ; BL = 7
div bl               ; AL = quotient, AH = remainder
```

**Result:** 100 ÷ 7 = 14 remainder 2 → AX = `0x020E`
- **AL = 0x0E = 14** (quotient)
- **AH = 0x02 = 2** (remainder)

**GDB showed:**
- Before `div`: `eflags 0x202 [ IF ]`
- After `div`: `eflags 0x202 [ IF ]`

| Flag | Status | Why |
|------|--------|-----|
| CF | 0 (**undefined**) | DIV doesn't use CF. Nothing changed. |
| OF | 0 (**undefined**) | DIV doesn't use OF. If the quotient were too big for AL, the CPU would raise a divide error instead of setting OF. |
| ZF | 0 (**undefined**) | Even though it's 0, it doesn't mean "the result isn't zero". It just wasn't touched. |
| SF, PF, AF | 0 (**undefined**) | Not meaningful after DIV. |

On my CPU, EFLAGS was exactly the same before and after the division. Everything stayed where the earlier `mov` instructions left it (and `mov` never changes flags). This shows that DIV didn't report anything through the flags.

---

## Program 2: `div2.asm` (16-bit)

```asm
mov ax, [dividend]   ; AX = 50000
mov dx, [highpart]   ; DX = 0
mov bx, [divisor]    ; BX = 300
div bx               ; AX = quotient, DX = remainder
```

**Result:** DX:AX = 50000. 50000 ÷ 300 = 166 remainder 200
- **AX = 0x00A6 = 166** (quotient)
- **DX = 0x00C8 = 200** (remainder)

**GDB showed:**
- Before `div`: `eflags 0x202 [ IF ]`
- After `div`: `eflags 0x202 [ IF ]`

| Flag | Status | Why |
|------|--------|-----|
| CF, OF, SF, ZF, AF, PF | all 0 (**undefined**) | Same as program 1. DIV doesn't use the flags to describe its result, so the flags look the same as before. |

**Why `mov dx, 0` matters:** DIV always uses DX:AX as the number to divide, not just AX. If DX had leftover junk in it, we'd be dividing a much bigger number than 50000, and the quotient might not fit in AX. That would crash the program with a divide error. Clearing DX first is the "safety step" for 16-bit division.

### Quick summary

| Instruction | Do the flags describe the result? |
|-------------|----------------------------------|
| ADD / SUB | Yes, all six (CF, ZF, SF, OF, PF, AF) |
| MUL | Only CF and OF (does the upper half have anything in it?) |
| DIV | None. Errors crash the program instead. |
