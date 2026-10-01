# SUB – Flags after subtraction

I ran `sub1.asm`, `sub2.asm` and `sub3.asm` in GDB and checked `info registers eflags` straight after each `sub` line.

(As in the add folder: IF is always on because Linux sets it, and arithmetic doesn't touch it. Also, the ZF you see at the very end comes from `xor ebx, ebx`, not from the subtraction.)

## How subtraction sets the flags

With SUB, **CF means "borrow"**. It turns on when the number you take away is bigger than the number you start with, comparing them as unsigned numbers. **OF** asks a different question: is the signed answer wrong?

## How to build and run

```bash
nasm -f elf32 -g sub1.asm -o sub1.o
ld -m elf_i386 sub1.o -o sub1
gdb ./sub1
(gdb) break _start
(gdb) run
(gdb) stepi            # until the sub has run
(gdb) info registers eflags
```

---

## Program 1: `sub1.asm` (8-bit)

```asm
mov al, [num1]   ; AL = 50
sub al, [num2]   ; AL = 50 - 80
```

**Result:** AL = `0xE2` = `1110 0010`. As a signed number that's **-30**, which is the right answer. As unsigned it reads as 226.

**GDB showed:** `eflags 0x287 [ CF PF SF IF ]`

| Flag | Status | Why |
|------|--------|-----|
| CF (Carry/Borrow) | **1 – set** | 50 is smaller than 80, so the CPU had to borrow from a bit that doesn't exist. As unsigned numbers, 50 - 80 has no correct answer, and CF warns you about that. |
| ZF | **0 – cleared** | The answer is -30, not zero. |
| SF | **1 – set** | Bit 7 of `1110 0010` is 1, so the result is negative. That matches -30. |
| OF | **0 – cleared** | Positive minus positive can never go outside the signed range (-128 to 127), and -30 fits. The signed answer is correct, so there's no overflow. |
| PF | **1 – set** | `1110 0010` has four 1s. Four is even. |
| AF | **0 – cleared** | Low nibbles: 50 ends in `0010` (2) and 80 ends in `0000` (0). 2 - 0 = 2, so no borrow was needed from bit 4. |

**Main lesson:** here CF = 1 but OF = 0. If you think of the numbers as unsigned, the result is wrong (borrow). If you think of them as signed, the result (-30) is right. That's the opposite of `add1.asm`, where OF was 1 and CF was 0.

---

## Program 2: `sub2.asm` (16-bit)

```asm
mov ax, [num1]   ; AX = 1000
sub ax, [num2]   ; AX = 1000 - 2000
```

**Result:** AX = `0xFC18`, which is **-1000** as a signed number.

**GDB showed:** `eflags 0x287 [ CF PF SF IF ]`

| Flag | Status | Why |
|------|--------|-----|
| CF | **1 – set** | 1000 is less than 2000, so there was a borrow out of bit 15. |
| ZF | **0 – cleared** | The answer is not zero. |
| SF | **1 – set** | Bit 15 of `0xFC18` is 1, so the result is negative (-1000). |
| OF | **0 – cleared** | -1000 easily fits in a signed 16-bit number (-32768 to 32767), so the signed answer is correct. |
| PF | **1 – set** | PF only checks the **low byte**, which is `0x18` = `0001 1000`. That has two 1s, which is even. |
| AF | **0 – cleared** | 1000 = `0x03E8` ends in 8, and 2000 = `0x07D0` ends in 0. 8 - 0 needs no borrow. |

**Main lesson:** this is the same pattern as program 1, but in 16 bits. It shows that PF only looks at the lowest 8 bits, even when the register is bigger.

---

## Program 3 (bonus): `sub3.asm` (SUB then SBB)

```asm
mov ax, [num1]   ; AX = 0
sub ax, [num2]   ; AX = 0 - 1
sbb ax, 0        ; AX = AX - 0 - CF
```

### After `sub ax, [num2]`

**Result:** AX = `0xFFFF` (-1 signed)

**GDB showed:** `eflags 0x297 [ CF PF AF SF IF ]`

| Flag | Status | Why |
|------|--------|-----|
| CF | **1 – set** | 0 - 1 needs a borrow, because 0 is smaller than 1. |
| ZF | **0 – cleared** | The answer isn't zero. |
| SF | **1 – set** | The top bit is 1, so the result is negative (-1). |
| OF | **0 – cleared** | 0 - 1 = -1 is the correct signed answer. |
| PF | **1 – set** | Low byte `1111 1111` has eight 1s, which is even. |
| AF | **1 – set** | The low nibble is 0 - 1, which also had to borrow from bit 4. |

### After `sbb ax, 0`

**Result:** AX = `0xFFFF` - 0 - CF(1) = `0xFFFE` (-2)

**GDB showed:** `eflags 0x282 [ SF IF ]`

SBB subtracts the borrow left over from the step before. It's the subtraction version of ADC.
- CF is now **0** because `0xFFFF` is bigger than 1, so no borrow was needed this time.
- SF stays **1** because the top bit is still 1 (the answer is -2).
- PF is **0** because `0xFE` = `1111 1110` has seven 1s, which is odd.
- AF is **0** because F - 1 = E, no borrow in the low nibble.
- ZF and OF are **0** because the answer isn't zero and -1 - 1 = -2 is a correct signed result.
