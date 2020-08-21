# UTF-8 encoding
UTF-8 encoding schema uses 1-4 bytes to encode a character, depending on its code point.

`c` represents the code point of a character, `e` represents the utf-8 encoded value of `c`, it's encoded as follows:

#### c < 0x80: _0*******_, 7 bits required
	
	e = c
	
#### c < 0x800: _110***** 10******_, 11 bits required

	b1 = (0xc0 | (c >> 6)) << 8
	b2 =  0x80 | (c & 0x3f)
	e = b1 | b2
	
#### c < 0x10000: _1110**** 10****** 10******_, 16 bits required

	b1 = (0xe0 | (c >> 12)) << 16
	b2 = (0x80 | ((c >> 6) & 0x3f)) << 8
	b3 =  0x80 | (c & 0x3f)
	e = b1 | b2 | b3

Note that here the surrogates have to be checked: high surrogates: [0xD800, 0xDC00), low surrogates: [0xDC00, 0xE000), the combination of which forms the code space [0x10000, 0x110000)

#### c < 0x110000: _11110*** 10****** 10****** 10******_

	b1 = (0xf0 | (c >> 18)) << 24
	b2 = (0x80 | ((c >> 12) & 0x3f)) << 16
	b3 = (0x80 | ((c >> 6) & 0x3f)) << 8
	b4 =  0x80 | (c & 0x3f)
	e = b1 | b2 | b3 | b4
