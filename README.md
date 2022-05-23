Hacking VTech 6502-based toys
=============================

This repository contains various pieces of information about the inside of some
VTech toys based around a 6502 chip. These are typically anonymous blobs, but
are probably a derivative of the GeneralPlus GPLB6X chip; this has some (the
amount is unknown) of internal mask ROM, about 2kB of RAM, a memory-mapped
interface allowing SPI flash to be used as ROM, and a CPU compatible with the
65c02: this is the CMOS version of the 6502 with the useful extra instructions
like `BRA` and `STZ`.

My Zone / Ordi Genius Kid
-------------------------

This is a little laptop thingy with a 64x32 2-bit screen, alphabetic ortho bubble keyboard, and d-pad.

![The My Zone](https://cdn-vtech-jouets.vtech.com/assets/7846f227-1252-4c68-b760-388aa10b1863/196305_ordi-genius-kid.jpg) 

Access was achieved by removing the surface-mount flash chip on the accessible
side of the motherboard, socketing it, and dumping the contents. This contains
the application code. There's an OS on ROM inside the blob IC itself which
provides useful functionality and which calls it.

Lots of things use three-byte pointers, where an address in high ROM is
followed by the bank number of the ROM.

Strings are terminated with ff. For some reason, fe bytes are ignored
inside strings.

You absolutely have to kick the watchdog at 0a36 or the machine will power
down. The timeout is very short.

### Memory map

```
0000-0aff   : RAM, stack, I/O
  0000-00ff : zero page
    00-02   
  0600-06ff : application workspace
    06e4    : string buffer; length unknown
  0a00-0aff : seems to be I/O
    0a36    : set bottom bit to kick the watchdog
    0a38    : high ROM bank register
0b00-7fff   : low ROM; this does not seem to be banked
8000-efff   : high ROM; banked. Banks >= 80 are SPI ROM, low numbers are mask ROM
f000-ffff   : unknown, maybe a fixed reset ROM; may not always be present?
```

The low ROM contains most useful operating system calls. Parameters are passed
in memory locations. Functions' use of parameters will overlap; the language
this was written in is probably packing them by examining the call graph.

 - 3988 Sound interrupt handler, maybe?

 - 6465 `clear_workspace`

   Wipes the application workspace and does other pieces of initialisation.

 - 65be `mul16`

   Multiplies the work in 0054 with the word in 0056 and puts the result in
   0054.

 - 6602 `play_sound`

   Plays a sound (in the background). 004c contains the sound number. This
   actually just jumps to `81:(0x8000 + soundnumber*0x19)`. Bank 81
   contains a huge array of identical routines which turns interrupts off,
   sets a three-byte pointer in 00-02 (to different values), and calls
   3988.

 - 725a `draw_sprite`

   Draws a sprite on the screen. 06d0-06d2 contains a three byte pointer to
   the sprite sheet; 06d3-06d4 contains the sprite number within the sheet;
   06d5/06d6 is X/Y.

   The sprite sheet is a simple array of three-byte pointers pointing at the
   actual sprite data.

   The sprite data itself has a three byte header (depth, height, width)
   followed by the picked bitmap. I've only experimented with 1-bit images.
   LSB is the _left_ pixel.

 - 752a `draw_string`

   Displays the string in the buffer at 06e4. This assumes that 83:8600
   contains a sprite sheet with the font in it, in ASCII-ish order with sprite
   0 corresponding to byte 20. Each sprite must be six pixels wide but I
   don't think it cares about the height. 06d5/06d6 is X/Y.

 - 753a `draw_string_centered`

   As `draw_string` but the string is centred horizontally. (X is ignored.)
 
 ### Application structure

 The OS expects the SPI flash to contain (at least) two tables at the beginning
 containing vectors:

```
80:8700 : array of one-byte banks for each vector
80:8800 : array of two-byte pointers for each vector
```

The vectors I've figured out so far are:

  - 9: startup program, which plays the opening jingle and asks for the user's
	name
  - a: shutdown program, which plays the shutdown animation and turns the
	machine off
  - b: diagnostics tool. I don't know how to invoke this deliberately, but
    patching its address into vector 9 will cause it to run on startup.

Why things are called so far remains a mystery...

