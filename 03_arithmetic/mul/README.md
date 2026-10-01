mul1.asm

25 * 10 = 250 (0x00FA) in AX.

CF = 0, OF = 0: the upper half (AH) is zero so the product fits in the 8-bit destination half.
SF, ZF, AF, PF: undefined by MUL; their values cannot be explained from the product.

mul2.asm

3000 * 200 = 600000 (0x000927C0) in DX:AX.

CF = 1, OF = 1: the upper half (DX = 9) is nonzero, so the product does not fit in AX alone.
SF, ZF, AF, PF: undefined by MUL; their values cannot be explained from the product.