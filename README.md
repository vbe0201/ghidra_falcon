# Nvidia Falcon plugin for Ghidra

This is pretty immature, so you'll probably need to do some development to use it, but it's likely good enough to save a bit of time over an `envydis` dead-listing.

![Screenshot](/images/screenshot1.png)

## Setup instructions

- Ensure you have the ``JAVA_HOME`` environment variable set to the path of your JDK 11 installation.
- Set the ``GHIDRA_INSTALL_DIR`` environment variable to your Ghidra install directory.
- Run ``./gradlew``.
- You'll find the output zip file inside ``/dist``.
- Copy the output zip to ``<Ghidra install directory>/Extensions/Ghidra``
- Start Ghidra and use the "Install Extensions" dialog to finish the installation. (``File -> Install Extensions...``)

## Development

This is just how I do it:

Open up the [envytools Falcon ISA documentation](https://envytools.readthedocs.io/en/latest/hw/falcon/isa.html) to compare the semantic information with the reference pseudocode. Open Ghidra's `doc/languages/index.html` for a reference for how the Sleigh language works.

Load your test binary in Ghidra.

Use [envydis](https://github.com/envytools/envytools) on the binary you're looking at so you can quickly check the disassembly for errors if anything seems off.

In Ghidra enable PCode display by clicking the "Edit the Listing fields" icon at the top of the Listing view, right clicking "PCode" and clicking "Enable Field" (you may wish to toggle this on-and-off during reversing as it is quite verbose).

In Ghidra open the script manager ("Window" -> "Script Manager") and add key-bindings for `DebugSleighInstructionParse.java` and `ReloadSleighLangauge.java`.

`DebugSleighInstructionParse.java` will log detailed debug information about the instruciton parse for the byte under the cursor. I usually use it to find the `{line# 756} ld_b32 <reg2>, D[<reg1>]` so I know what line to go to to fix the PCode output.

`ReloadSleighLangauge.java` will reload most changes to the slaspec, including new instructions and new semantic definitions. However, if you add new registers or new `define pcodeop ...` statements, you will need to restart Ghidra to get correct output.

### Adding instructions to the decompiler

At the moment any particularly poorly implemented instruction should show up as a call to the `todo` pcodeop in the decompiler output - this is a hint that the instruction should be implemented for correct decompilation. Get the line number using `DebugSleighInstructionParse.java` on the corresponding instruction, and add the implementation to the slaspec/sinc file.

See Ghidra's `doc/languages/index.html` for more information about how to describe instructions.

Run `ReloadSleighLangauge.java`, or, if you've added a new pcodeop for the instruction, close and re-open Ghidra to see the new output.


### Adding instructions to the disassembler

If you hit undecodable instruction bytes, find the correct decoding in your `envydis` output, then search through the `envydis.sinc` file for the envydis source comment that matches. For example:

```
#	{ 0x0004003c, 0x000f003f, OP3B, N("shl"), T(sz), REG3, REG1, REG2 },
```

Try to find an implemented instruction using the same encoding/operands, for example:

```
#	{ 0x0002003c, 0x000f003f, OP3B, N("sub"), T(sz), REG3, REG1, REG2 },
:sub_b32 reg3, reg1, reg2 is op_format=0x3c & op_size=2; reg2 & reg1; subopcode3=0x2 & reg3 {
  sub(reg3, reg1, reg2);
}
```

Copy-paste and change the opcode/subopcode, mnemonic, and implementation:

```
:shl_b32 reg3, reg1, reg2 is op_format=0x3c & op_size=2; reg2 & reg1; subopcode3=0x4 & reg3 {
  todo();
}
```

Reload (with your `ReloadSleighLangauge.java` hotkey), disassemble the bytes (by pressing `D`), and check the output is correct. If it fails to disassemble, double check your code, or use `DebugSleighInstructionParse.java` to see where it goes wrong.

If you hit errors reloading, you can view the log by clicking the "Show Console" icon at the bottom Ghidra project window (the one with the file listing, not the CodeBrowser/disassembly window). The errors are only helpful some of the time, so I usually check my changes by hand first to see if I can spot what I did wrong.

### Syntax highlighting

If you like syntax highlighting, Sleigh often looks okay highlighted as a bash script. I got tired of this and wrote a very basic sublime-syntax, which can be found at: https://github.com/hthh/sleigh-sublime-syntax

## To-Do

Finish adding instructions and figure out how to do thorough automated testing.

It should be practical to generate a huge test file covering every instruction encoding with different operand values, and automatically compare the Ghidra and envydis output to finish the instruction decoding.

No idea how best to test the decompilation/pcode.

Known issues/todo-list:

* add other falcon variants
* fix mpop/mpush (and variants) to read/write all values
* rename "ram" to "iram" and document adding a "dram" section for globals
* try making "io" a real address space? might make naming easier
* see if there's a way to make crypto stuff decompile as `csecret(c1, 0x2)` instead of `csecret(1, 2)`
* global stores can be reordered to come after pcodeop calls that implicitly read from them (probably a ghidra bug?)
* can we bundle types? I'm loading an enum with io address constants (e.g. `FALCON_MAILBOX1 = 0x1100`) to make reversing easier. ("io" address space and default names might be the right approach?)
* finish adding packflags and unpackflags anywhere the flags register is used directly. make "xbit" on condition flags just move the one register.
* make the flags register actually show up when not just used for condition codes. writes to flags such as  "interrupt enable" don't show up in the decompiler.
* update readme screenshot sometime - that function is looking a ton better already (https://imgur.com/KGQJebQ), but I want to try to get things stable first to avoid making the repo huge with png files

## Resources

* https://envytools.readthedocs.io/en/latest/hw/falcon/index.html - thorough hardware documentation
* https://switchbrew.org/wiki/TSEC - crypt functionality
* http://0x04.net/~mwk/Falcon.html - calling convention
