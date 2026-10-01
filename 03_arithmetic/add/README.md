add1.asm

120 + 10 = 130 (0x82) in an 8-bit destination.

CF = 0: the unsigned result fits in 8 bits; there is no carry out.
OF = 1: the signed operands are positive, but the 8-bit result has its sign bit set.
SF = 1: the result's most significant bit is 1.
ZF = 0: the result is nonzero.
PF = 1: `0x82` has two set bits, an even number of 1 bits in the low byte.
AF = 0: adding the low nibbles (`8 + 0`) produces no carry from bit 3 to bit 4.

add2.asm

32000 + 500 = 32500 (0x7F24) in a 16-bit destination.

CF = 0: the unsigned result fits in 16 bits.
OF = 0: the signed result fits in the 16-bit signed range.
SF = 0: the result's most significant bit is 0.
ZF = 0: the result is nonzero.
PF = 1: the low byte, `0x24`, has two set bits (even parity).
AF = 0: the low nibbles add without a carry from bit 3 to bit 4.