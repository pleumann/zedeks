# Tapster #

Tapster is a little BASIC tool that allows you to easily and automatically transfer tapes to the Next's disk. It does not create a TAP file, but individual files that can easily be edited later. So this is not meant for getting all your tapes full of games to the Next (use [HeartWare's Tape2Tap](https://www.facebook.com/groups/specnext/permalink/1154746544882664/) for that), but rather for tapes that contain BASIC programs from back-in-the-day, pictures, Tasword files or other things you might want to "migrate" to your lovely new machine and continue using or even improving there. In that sense I consider it a spiritual successor to the converter tool that came on the introductory Sinclair Microdrive cartridge.

![Screenshots](https://github.com/pleumann/zxstuff/blob/master/tapster/tapster.png?raw=true)

File size is normally limited to around 32K. You can move the program into a memory bank to get around 40K of available space, allowing larger files to be processed:

```
BANK NEW b
BANK b LINE 0,9999
ERASE 0,9999
BANK b goto 10
```

Using Tapster is as easy as selecting a target directory (notice that unlike the usual case you have to confirm with the *space bar* when in the Next's file browser) and selecting "Start". Several options allow you to fine-tune the conversion process:

* Duplicate files can be kept (by numbering them) or overwrite previous ones of the same name.
* File names can be kept as-is or modified to follow the rules of the NextZXOS file system.
* Suffixes (such as ```.bas``` or ```.scr```) can be generated, so the browser knows what type of file it is.
* Headerless file can be allowed, although these will always be saved as ```"Untitled" CODE```.

A logfile ```tapster.txt``` is automatically created in the target directory (including timestamp if you have an RTC in your Next). This allows you to import a full C60 or C90 in an unattended manner and check the results afterwards to see which files were converted and what their final names were.

## How does it work? ##

For the fun of it, let's have a look at how Tapster works. At the heart of the program are two little machine code subroutines. In fact they are so tiny I was too lazy to fire up a proper Assembler and preferred to ```POKE``` them into memory instead (using Appendix A of the excellent Next User Manual as a reference).

The first routine loads a block from tape. For an ordinary file this routine is called twice, first for the file's header, then for the file's data. The length of the block (including a leading byte and a trailing checksum) is returned in the BC register pair, so we can assign it to a variable in BASIC. We then analyze the header according to the [tape specification](https://faqwiki.zxnet.co.uk/wiki/Spectrum_tape_interface#Blocks) in order to know what we are dealing with and print that to the screen.

```
        LD IX,start
        LD DE,length
        LD C,$0E
        SCF
        EX AF,AF'
        DI
        PUSH DE
        CALL $056C
        EI
        POP HL
        SCF
        SBC HL,DE
        PUSH HL
        POP BC
        INC BC
        RET
```

The second piece of machine code generates a checksum for a block in memory. We need two different kinds of checksums. Tape checksums are an XOR of all bytes in a block. Disk checksums are the sum of all bytes in the file's +3DOS header (or the first 127 bytes of it, to be correct) modulo 256. The routine generates both at the same time and returns them in the upper and lower halves of BC, respectively.

```
        LD HL,start
        LD DE,length
        LD BC,0
LOOP    LD A,D
        OR A,E
        RETZ
        LD A,B
        ADD A,(HL)
        LD B,A
        LD A,C
        XOR (HL)
        LD C,A
        INC HL
        DEC DE
        JR LOOP
```

For creating the disk file we need to apply a little bit of a trick. A BASIC program cannot easily load another BASIC program and then save that second program to disk (same for numeric or character arrays). Also we don't even know what we're loading before we start doing it. Hence we always save the tape data as ```CODE``` in the first step, which already gives us a disk file with proper +3DOS header, length and content for free. Then, in a second step, we simply patch the file's +3DOS header (according to [the Spectrum +3 manual, chapter 27](https://k1.spdns.de/Vintage/Sinclair/86/ZX%20Spectrum%2B3/ZX%20Spectrum%2B3%20Manual/chapter8pt27.html)) to make sure it has the correct type and parameters. Before writing the changed header back to disk we have to regenerate the checksum, and we're done. Amazingly this can all be done in BASIC. :)

## The Next level ##

This was my first ZX Spectrum program (we don't call these apps, do we?) after 30 years, so I used it as a playground for exploring some of the new features that are part of the enhanced NextBASIC. The following I found useful (in no particular order):

* ```DEFPROC```, ```PROC``` and ```LOCAL``` help a lot to structure the code (although there is still a bit of spaghetti in there, but that's not the Next's fault).
* I used a text window channel with narrow font for the lower part of the screen. Assigning it to ```#3```meant I could enjoy the brevity of just ```LPRINT```.
* The tape needs to be loaded at 3.5 MHz, but the saving and checksumming should ideally be done at 28 MHz so we don't miss the next tape block. Luckily, NextBASIC can switch between CPU speeds at any time using ```RUN AT```.
* The various new variants of ```PEEK``` and ```POKE``` resulted in code that was both shorter and more readable. No more ```FOR ... NEXT``` loops for machine code or UDGs. Yay!
* ```ON ERROR``` is a nice way of handling errors. With the nesting it feels a bit like exceptions in Java or other grown-up languages. Being able to access the actual error message (not just the code) would be nice.
* The ```.date``` and ```.time``` dot commands get, well, date and time. It currently needs a little trick to get this into a variable, though. A consistent mechanism for "piping" the output of a dot command into a variable would be a welcome improvement.
* The ```.browse``` dot command, last but not least, allows you to integrate the full-fledged system file browser into any of your programs. Cute!
