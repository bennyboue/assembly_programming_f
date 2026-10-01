sub1.asm

50 - 80 = -30, represented as 0xE2 in an 8-bit destination.

CF = 1: unsigned subtraction needs a borrow because 50 is less than 80.
OF = 0: -30 is within the signed 8-bit range (-128 through 127).
SF = 1: the result's most significant bit is 1.
ZF = 0: the result is nonzero.
PF = 1: 0xE2 has four set bits (even parity).
AF = 0: the low-nibble subtraction (0x2 - 0x0) needs no borrow from bit 4.

sub2.asm

1000 - 2000 = -1000, represented as 0xFC18 in a 16-bit destination.

CF = 1: unsigned subtraction needs a borrow because 1000 is less than 2000.
OF = 0: -1000 is within the signed 16-bit range (-32768 through 32767).
SF = 1: the result's most significant bit is 1.
ZF = 0: the result is nonzero.
PF = 1: the low byte, `0x18`, has two set bits (even parity).
AF = 0: the low-nibble subtraction (0x0 - 0x0) needs no borrow from bit 4.