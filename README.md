# NandToTetris_HDLcode
# Hack HDL Chip Set (nand2tetris)

Every chip is built from the single primitive **Nand**, so you never write `Nand` yourself. Each chip goes in its own file named after the chip, e.g. `Not.hdl`, `And.hdl`.

## Build order

```
Nand (built-in)
 ├─ Not ─ And ─ Or ─ Xor ─ Mux ─ DMux
 │
 └─ HalfAdder ─ FullAdder ─ Add16 (ripple carry)
                              │
 Not16, And16, Mux16, Or8Way ─┴─ ALU
```

Build and test in this order, because each chip uses the ones above it.

---

## 1. Not

**Idea:** A Nand gate with both inputs tied together. `Nand(x, x) = Not(x)`.

| in | out |
|----|-----|
| 0  | 1   |
| 1  | 0   |

```hdl
CHIP Not {
    IN in;
    OUT out;
    PARTS:
    Nand(a=in, b=in, out=out);
}
```

## 2. And

**Idea:** And is the opposite of Nand, so invert the Nand output.

| a | b | out |
|---|---|-----|
| 0 | 0 | 0   |
| 0 | 1 | 0   |
| 1 | 0 | 0   |
| 1 | 1 | 1   |

```hdl
CHIP And {
    IN a, b;
    OUT out;
    PARTS:
    Nand(a=a, b=b, out=nandab);
    Not(in=nandab, out=out);
}
```

## 3. Or

**Idea:** De Morgan's law: `a OR b = NOT(NOT a AND NOT b)`. Invert both inputs, then Nand them.

| a | b | out |
|---|---|-----|
| 0 | 0 | 0   |
| 0 | 1 | 1   |
| 1 | 0 | 1   |
| 1 | 1 | 1   |

```hdl
CHIP Or {
    IN a, b;
    OUT out;
    PARTS:
    Not(in=a, out=nota);
    Not(in=b, out=notb);
    Nand(a=nota, b=notb, out=out);
}
```

## 4. Xor

**Idea:** Output is 1 when the inputs differ: `(a AND NOT b) OR (NOT a AND b)`.

| a | b | out |
|---|---|-----|
| 0 | 0 | 0   |
| 0 | 1 | 1   |
| 1 | 0 | 1   |
| 1 | 1 | 0   |

```hdl
CHIP Xor {
    IN a, b;
    OUT out;
    PARTS:
    Not(in=a, out=nota);
    Not(in=b, out=notb);
    And(a=a, b=notb, out=w1);
    And(a=nota, b=b, out=w2);
    Or(a=w1, b=w2, out=out);
}
```

## 5. Mux (multiplexer)

**Idea:** A selector. `sel = 0` passes `a`, `sel = 1` passes `b`.
Formula: `out = (a AND NOT sel) OR (b AND sel)`.

| a | b | sel | out |
|---|---|-----|-----|
| 0 | x | 0   | 0   |
| 1 | x | 0   | 1   |
| x | 0 | 1   | 0   |
| x | 1 | 1   | 1   |

```hdl
CHIP Mux {
    IN a, b, sel;
    OUT out;
    PARTS:
    Not(in=sel, out=notsel);
    And(a=a, b=notsel, out=w1);
    And(a=b, b=sel, out=w2);
    Or(a=w1, b=w2, out=out);
}
```

## 6. DMux (demultiplexer)

**Idea:** The reverse of Mux. It sends `in` to output `a` when `sel = 0`, or to output `b` when `sel = 1`. The other output stays 0.

| in | sel | a | b |
|----|-----|---|---|
| 0  | 0   | 0 | 0 |
| 0  | 1   | 0 | 0 |
| 1  | 0   | 1 | 0 |
| 1  | 1   | 0 | 1 |

```hdl
CHIP DMux {
    IN in, sel;
    OUT a, b;
    PARTS:
    Not(in=sel, out=notsel);
    And(a=in, b=notsel, out=a);
    And(a=in, b=sel, out=b);
}
```

---

## 7. HalfAdder

**Idea:** Adds two bits. `sum = a XOR b`, `carry = a AND b`.

| a | b | carry | sum |
|---|---|-------|-----|
| 0 | 0 | 0     | 0   |
| 0 | 1 | 0     | 1   |
| 1 | 0 | 0     | 1   |
| 1 | 1 | 1     | 0   |

```hdl
CHIP HalfAdder {
    IN a, b;
    OUT sum, carry;
    PARTS:
    Xor(a=a, b=b, out=sum);
    And(a=a, b=b, out=carry);
}
```

## 8. FullAdder (2 HalfAdders + Or)

**Idea:** Adds three bits (`a`, `b`, and a carry-in `c`).

1. The first HalfAdder adds `a + b`, giving a partial sum `s1` and carry `c1`.
2. The second HalfAdder adds `s1 + c`, giving the final `sum` and carry `c2`.
3. A carry out happens if **either** half adder carried, so `carry = c1 OR c2`. (Both can never carry at once.)

| a | b | c | carry | sum |
|---|---|---|-------|-----|
| 0 | 0 | 0 | 0     | 0   |
| 0 | 0 | 1 | 0     | 1   |
| 0 | 1 | 0 | 0     | 1   |
| 0 | 1 | 1 | 1     | 0   |
| 1 | 0 | 0 | 0     | 1   |
| 1 | 0 | 1 | 1     | 0   |
| 1 | 1 | 0 | 1     | 0   |
| 1 | 1 | 1 | 1     | 1   |

```hdl
CHIP FullAdder {
    IN a, b, c;
    OUT sum, carry;
    PARTS:
    HalfAdder(a=a, b=b, sum=s1, carry=c1);
    HalfAdder(a=s1, b=c, sum=sum, carry=c2);
    Or(a=c1, b=c2, out=carry);
}
```

## 9. Ripple Carry Adder (Add16)

**Idea:** Chain 16 adders. The carry-out of each bit becomes the carry-in of the next, so the carry "ripples" from bit 0 to bit 15.

- Bit 0 has no carry-in, so it uses a HalfAdder.
- Bits 1 to 15 use FullAdders.
- The final carry (`c15`) is dropped, because the Hack chip ignores overflow.

```hdl
CHIP Add16 {
    IN a[16], b[16];
    OUT out[16];
    PARTS:
    HalfAdder(a=a[0],  b=b[0],  sum=out[0],  carry=c0);
    FullAdder(a=a[1],  b=b[1],  c=c0,  sum=out[1],  carry=c1);
    FullAdder(a=a[2],  b=b[2],  c=c1,  sum=out[2],  carry=c2);
    FullAdder(a=a[3],  b=b[3],  c=c2,  sum=out[3],  carry=c3);
    FullAdder(a=a[4],  b=b[4],  c=c3,  sum=out[4],  carry=c4);
    FullAdder(a=a[5],  b=b[5],  c=c4,  sum=out[5],  carry=c5);
    FullAdder(a=a[6],  b=b[6],  c=c5,  sum=out[6],  carry=c6);
    FullAdder(a=a[7],  b=b[7],  c=c6,  sum=out[7],  carry=c7);
    FullAdder(a=a[8],  b=b[8],  c=c7,  sum=out[8],  carry=c8);
    FullAdder(a=a[9],  b=b[9],  c=c8,  sum=out[9],  carry=c9);
    FullAdder(a=a[10], b=b[10], c=c9,  sum=out[10], carry=c10);
    FullAdder(a=a[11], b=b[11], c=c10, sum=out[11], carry=c11);
    FullAdder(a=a[12], b=b[12], c=c11, sum=out[12], carry=c12);
    FullAdder(a=a[13], b=b[13], c=c12, sum=out[13], carry=c13);
    FullAdder(a=a[14], b=b[14], c=c13, sum=out[14], carry=c14);
    FullAdder(a=a[15], b=b[15], c=c14, sum=out[15], carry=c15);
}
```

---

## 10. Helper chips the ALU needs

These apply the 1-bit gates to every bit of a 16-bit bus.

### Not16
```hdl
CHIP Not16 {
    IN in[16];
    OUT out[16];
    PARTS:
    Not(in=in[0],  out=out[0]);
    Not(in=in[1],  out=out[1]);
    Not(in=in[2],  out=out[2]);
    Not(in=in[3],  out=out[3]);
    Not(in=in[4],  out=out[4]);
    Not(in=in[5],  out=out[5]);
    Not(in=in[6],  out=out[6]);
    Not(in=in[7],  out=out[7]);
    Not(in=in[8],  out=out[8]);
    Not(in=in[9],  out=out[9]);
    Not(in=in[10], out=out[10]);
    Not(in=in[11], out=out[11]);
    Not(in=in[12], out=out[12]);
    Not(in=in[13], out=out[13]);
    Not(in=in[14], out=out[14]);
    Not(in=in[15], out=out[15]);
}
```

### And16
```hdl
CHIP And16 {
    IN a[16], b[16];
    OUT out[16];
    PARTS:
    And(a=a[0],  b=b[0],  out=out[0]);
    And(a=a[1],  b=b[1],  out=out[1]);
    And(a=a[2],  b=b[2],  out=out[2]);
    And(a=a[3],  b=b[3],  out=out[3]);
    And(a=a[4],  b=b[4],  out=out[4]);
    And(a=a[5],  b=b[5],  out=out[5]);
    And(a=a[6],  b=b[6],  out=out[6]);
    And(a=a[7],  b=b[7],  out=out[7]);
    And(a=a[8],  b=b[8],  out=out[8]);
    And(a=a[9],  b=b[9],  out=out[9]);
    And(a=a[10], b=b[10], out=out[10]);
    And(a=a[11], b=b[11], out=out[11]);
    And(a=a[12], b=b[12], out=out[12]);
    And(a=a[13], b=b[13], out=out[13]);
    And(a=a[14], b=b[14], out=out[14]);
    And(a=a[15], b=b[15], out=out[15]);
}
```

### Mux16
One `sel` bit controls all 16 Mux gates. `sel = 0` picks `a`, `sel = 1` picks `b`.
```hdl
CHIP Mux16 {
    IN a[16], b[16], sel;
    OUT out[16];
    PARTS:
    Mux(a=a[0],  b=b[0],  sel=sel, out=out[0]);
    Mux(a=a[1],  b=b[1],  sel=sel, out=out[1]);
    Mux(a=a[2],  b=b[2],  sel=sel, out=out[2]);
    Mux(a=a[3],  b=b[3],  sel=sel, out=out[3]);
    Mux(a=a[4],  b=b[4],  sel=sel, out=out[4]);
    Mux(a=a[5],  b=b[5],  sel=sel, out=out[5]);
    Mux(a=a[6],  b=b[6],  sel=sel, out=out[6]);
    Mux(a=a[7],  b=b[7],  sel=sel, out=out[7]);
    Mux(a=a[8],  b=b[8],  sel=sel, out=out[8]);
    Mux(a=a[9],  b=b[9],  sel=sel, out=out[9]);
    Mux(a=a[10], b=b[10], sel=sel, out=out[10]);
    Mux(a=a[11], b=b[11], sel=sel, out=out[11]);
    Mux(a=a[12], b=b[12], sel=sel, out=out[12]);
    Mux(a=a[13], b=b[13], sel=sel, out=out[13]);
    Mux(a=a[14], b=b[14], sel=sel, out=out[14]);
    Mux(a=a[15], b=b[15], sel=sel, out=out[15]);
}
```

### Or8Way
Output is 1 if **any** of the 8 input bits is 1.
```hdl
CHIP Or8Way {
    IN in[8];
    OUT out;
    PARTS:
    Or(a=in[0], b=in[1], out=o01);
    Or(a=o01,   b=in[2], out=o012);
    Or(a=o012,  b=in[3], out=o0123);
    Or(a=o0123, b=in[4], out=o4);
    Or(a=o4,    b=in[5], out=o5);
    Or(a=o5,    b=in[6], out=o6);
    Or(a=o6,    b=in[7], out=out);
}
```

---

## 11. ALU

**Inputs:** two 16-bit numbers `x`, `y` and six control bits.
**Outputs:** the 16-bit result `out`, plus two flags: `zr` (result is zero) and `ng` (result is negative).

The control bits are applied in this order:

| Step | Control | If 1                 |
|------|---------|----------------------|
| 1    | `zx`    | set x to 0           |
| 2    | `nx`    | invert x (bitwise)   |
| 3    | `zy`    | set y to 0           |
| 4    | `ny`    | invert y (bitwise)   |
| 5    | `f`     | `x + y` (else `x & y`) |
| 6    | `no`    | invert the output    |

```hdl
CHIP ALU {
    IN x[16], y[16], zx, nx, zy, ny, f, no;
    OUT out[16], zr, ng;
    PARTS:
    // x: zero it, then optionally negate
    Mux16(a=x, b=false, sel=zx, out=x1);
    Not16(in=x1, out=notx1);
    Mux16(a=x1, b=notx1, sel=nx, out=x2);

    // y: zero it, then optionally negate
    Mux16(a=y, b=false, sel=zy, out=y1);
    Not16(in=y1, out=noty1);
    Mux16(a=y1, b=noty1, sel=ny, out=y2);

    // f: 0 -> And, 1 -> Add
    And16(a=x2, b=y2, out=andxy);
    Add16(a=x2, b=y2, out=addxy);
    Mux16(a=andxy, b=addxy, sel=f, out=fout);

    // optionally negate output; also expose sign bit and two halves for zr
    Not16(in=fout, out=notfout);
    Mux16(a=fout, b=notfout, sel=no, out=out, out[15]=ng, out[0..7]=low, out[8..15]=high);

    // zr = 1 when all 16 bits are 0
    Or8Way(in=low, out=orlow);
    Or8Way(in=high, out=orhigh);
    Or(a=orlow, b=orhigh, out=nonzero);
    Not(in=nonzero, out=zr);
}
```

### How the flags work
- **`ng`** is the top bit (bit 15) of the result. In two's complement, that bit is 1 for negative numbers.
- **`zr`** is 1 only when every bit is 0. The result is split into two bytes, each is OR-reduced with `Or8Way`, the two results are ORed, and the answer is inverted.
- Writing `out=out, out[15]=ng, ...` on the same pin lets you send the full bus to the output and also tap parts of it internally.

### Common ALU operations (for testing)

| zx nx zy ny f no | Result |
|------------------|--------|
| 1 0 1 0 1 0      | 0      |
| 1 1 1 1 1 1      | 1      |
| 1 1 1 0 1 0      | -1     |
| 0 0 1 1 0 0      | x      |
| 1 1 0 0 0 0      | y      |
| 0 0 1 1 0 1      | !x     |
| 0 0 1 1 1 1      | -x     |
| 0 0 0 0 1 0      | x + y  |
| 0 1 0 0 1 1      | x - y  |
| 0 0 0 0 0 0      | x & y  |
| 0 1 0 1 0 1      | x \| y |

---

## Testing tips

- Load each `.hdl` file in the Hardware Simulator and run the matching `.tst` script with its `.cmp` file.
- If a chip fails, check pin names first (`a`, `b`, `sel`, `in`, `out`), then check for missing semicolons.
- Internal wire names (like `nota`, `w1`, `c0`) are arbitrary, but each one must be **fed by exactly one output**.
- In HDL, `false` and `true` are built-in constant buses.
