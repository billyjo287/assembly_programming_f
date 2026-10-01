# ADD – Flags after addition

I ran `add1.asm`, `add2.asm` and `add3.asm` in GDB. For each one I stopped right after the `add` line and checked the flags with `info registers eflags`.

**Quick tip:** check the flags straight after the `add`. If you wait until the end of the program, you will see ZF and PF set, but that comes from `xor ebx, ebx` in the exit code, not from the addition.

**About IF:** IF (Interrupt Flag) shows up in every run. Linux turns it on so the CPU can respond to interrupts. Arithmetic never changes it, so I leave it out below.

## How to build and run

```bash
nasm -f elf32 -g add1.asm -o add1.o
ld -m elf_i386 add1.o -o add1
gdb ./add1
(gdb) break _start
(gdb) run
(gdb) stepi            # repeat until the add has run
(gdb) info registers eflags
```

---

## Program 1: `add1.asm` (8-bit)

```asm
mov al, [num1]   ; AL = 120
add al, [num2]   ; AL = 120 + 10
```

**Result:** AL = 130 = `0x82` = `1000 0010`

**GDB showed:** `eflags 0xa96 [ PF AF SF IF OF ]`

| Flag | Status | Why |
|------|--------|-----|
| CF (Carry) | **0 – cleared** | Unsigned, 130 still fits in one byte, because a byte can hold up to 255. Nothing spilled out of the top bit. |
| ZF (Zero) | **0 – cleared** | The answer is 130, not zero. |
| SF (Sign) | **1 – set** | The top bit (bit 7) of `1000 0010` is 1. The CPU treats a top bit of 1 as "negative". |
| OF (Overflow) | **1 – set** | This is the interesting one. A signed byte can only go up to +127. We added two positive numbers (120 and 10) and the true answer, 130, is too big, so the result wrapped around and now looks negative (-126). Positive + positive = negative is exactly what OF is there to catch. |
| PF (Parity) | **1 – set** | PF only looks at the low byte. `1000 0010` has two 1s, and two is even, so PF = 1. |
| AF (Auxiliary carry) | **1 – set** | Look at the low 4 bits only. 120 ends in `1000` (8) and 10 ends in `1010` (10). 8 + 10 = 18, which is bigger than 15, so a carry went from bit 3 into bit 4. |

**Main lesson:** CF and OF can disagree. As **unsigned** numbers the answer (130) is fine, so CF = 0. As **signed** numbers the answer is wrong (it shows -126), so OF = 1. The CPU doesn't know which one you meant, so it sets both flags and leaves it to you to check the right one.

---

## Program 2: `add2.asm` (16-bit)

```asm
mov ax, [num1]   ; AX = 32000
add ax, [num2]   ; AX = 32000 + 500
```

**Result:** AX = 32500 = `0x7EF4` = `0111 1110 1111 0100`

**GDB showed:** `eflags 0x202 [ IF ]` (all arithmetic flags are 0)

| Flag | Status | Why |
|------|--------|-----|
| CF | **0 – cleared** | 32500 fits easily in 16 bits (max 65535). No carry out of bit 15. |
| ZF | **0 – cleared** | The answer is not zero. |
| SF | **0 – cleared** | Bit 15 is 0, so the result counts as positive. |
| OF | **0 – cleared** | A signed 16-bit number goes up to +32767. 32500 is just under that, so the signed answer is still correct. (If num2 were 800 instead of 500, we would get 32800, which is over the limit, and OF would turn on.) |
| PF | **0 – cleared** | The low byte is `0xF4` = `1111 0100`, which has five 1s. Five is odd, so PF = 0. |
| AF | **0 – cleared** | Low nibbles: 32000 ends in `0000` and 500 ends in `0100`. 0 + 4 = 4, no carry out of bit 3. |

**Main lesson:** we're close to the signed limit, but we don't cross it, so every flag stays off. This shows that the flags react to the actual result, not to how big the numbers look.

---

## Program 3 (bonus): `add3.asm` (ADD then ADC)

```asm
mov ax, [num1]   ; AX = 0xFFFF (65535)
add ax, [num2]   ; AX = 0xFFFF + 1
adc ax, 0        ; AX = AX + 0 + CF
```

### After `add ax, [num2]`

**Result:** AX = `0x0000`

**GDB showed:** `eflags 0x257 [ CF PF AF ZF IF ]`

| Flag | Status | Why |
|------|--------|-----|
| CF | **1 – set** | 65535 + 1 = 65536, which needs 17 bits. The extra 1 fell out of the top of AX and landed in CF. That's why AX became 0. |
| ZF | **1 – set** | What's left in AX is exactly 0. |
| SF | **0 – cleared** | The top bit of 0 is 0. |
| OF | **0 – cleared** | As signed numbers, `0xFFFF` is -1. -1 + 1 = 0, which is the correct answer, so there's no signed overflow. |
| PF | **1 – set** | Low byte `0000 0000` has zero 1s, and zero counts as even. |
| AF | **1 – set** | Low nibble: F + 1 = 16, so a carry left bit 3. |

### After `adc ax, 0`

**Result:** AX = 0 + 0 + CF(1) = `0x0001`

**GDB showed:** `eflags 0x202 [ IF ]`

ADC adds the carry from the step before. This is how you add numbers that are bigger than one register: the carry that got lost in the first add is put back here. 1 is a small, positive, non-zero number with an odd count of 1-bits and no carries, so CF, ZF, SF, OF, PF and AF are all cleared.
