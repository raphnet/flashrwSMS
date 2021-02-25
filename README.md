# Cartridge FLASH read/write code for Master System / Game Gear

raphnet has released SMS and Game Gear reprogrammable cartridge
circuit boards based on the MX29F040 FLASH chip. See the
[Project/Design notes page](https://www.raphnet.net/electronique/sms_gg_cartridge_pcb/index_en.php)
for more information.

Since the FLASH is rewritable, this design does not incorporate a
battery-backed RAM for saving scores and/or progress. These can
simply be recorded to FLASH using special code.

The code in this repository is intended for SMS or Game Gear developers
who need to write to the cartridge FLASH.

Currently supported chips:
 * MX29F040

Supported platforms:
 * SMS / SMS-J. Should also work on Mark-III systems.
 * Game Gear


## How this works

It is possible to make a cartridge with a single flash chip used
both for holding the game "ROM" and saving wihout requiring a
battery backed SRAM.

Such flash chips have a write pin and are wired like you would an SRAM,
but to enable operations such as programming and erasing, a specific
sequence of writes is required. For instance, one must do things such as
writing AA to 555, followed by 55 to 2AA, and so on. (See the code
or the flash datasheet for details).

Now, the actual code the console is running is stored on the flash. This
means that read accesses are performed as new instructions are fetched,
making it impossible to perform the required write sequences for initiating
various flash commands. The extraneous read cycles that would appear between the
required writes would abort the operation. Not to mention that during programming
and erasing, operations that require time, one cannot read from the flash as the
data bits then indicate the operation status rather than data at the requested address.

The solution is to run the code from elsewhere, and the only available place,
unless the cartridge had two flash chips or an additional ROM, is the console
built-in memory. This is achieved by copying machine code to memory, setting
up a function pointer and calling it.

Slot 2 is used as a 16k window for all flash accesses. The machine code takes care
of bank switching automatically and restores the original bank that was in slot 2
before returning. This means that the C code in this file may reside anywhere, even
in slot 2.

During flash operations, interrupts are disabled. Otherwise, if a vblank interrupt
were to occur, the CPU would try to read from the flash and the console would crash.
If one always performs flash operations after calling SMS_waitForVBlank() this should
not be necessary, but it's good to be safe.

The only problem I don't know how to solve happens if the user presses the PAUSE button
right in the middle of a flash operation. The pause button is, as far as I know, wired
to the NMI and cannot be disabled. But depending on the amount of data written,
the flash operation may very well complete in less that 1 ms. The odds of running into
this seem low enough. Also, unless the score is saved continuously (which is bad, see below),
the flash IO probably won't take place at a moment where a player is likely to pause the
game. (for instance, who pauses a game at the Game over screen?)

## Caution!

FLASH is not SRAM. It will wear down quickly written excessively to!

The MX29F040 is good for *at least* 100000 erase/program cycles,
but the point is that there is a limit, and if good practises
are not followed, there will be problems after a while...

### Recommendations / Good practises:

 * Do not write data to the flash continuously. For instance, do not write the score each time it increases. Do it once at game over. Or if it is a save function, do not autosave (unless rarely). Ideally, have the user do it from a menu or in-game save point.
 * Avoid writing when nothing has changed.
 * If you really want the flash to last as long as possible, implement a wear levelling algorithm. There are MANY ways to do that, but here is a simple example:
   * Use a fixed size structure for all your save data. Have this structure end by a fixed byte != 0xFF.
   * When saving, scan the flash sector backwards until the first != 0xFF byte is found to find out where the last structure was written.
   * Your startup code will have to scan the flash sector backwards to find the starting point, subtract the size of the fixed size structure and read the last saved data from there.
   * The above can even be made to spread across several sectors if you are really motivated.

### Other things that could be good to know:
 * Erased sectors will read 0xFF.
 * Unprogrammed byte (0xFF) can be programmed at any time.
 * I'm not sure, but I think this even works at the bit level. (i.e. you can turn 1s into 0s, but not the opposite without doing a sector erase)


## Supporting Flash and SRAM

It is possible to make a game that supports both standard SRAM (for emulator and "normal" cartridges) and FLASH. The FLASH chip can be auto-detected at start up, and if present, you use the flash read/write functions, otherwise use SRAM. See flash_isKnownId() and flash_readSiliconID() to see how auto-detection works.

While the code in flash.c can reside in bank 2, don't forget that your code
for accessing SRAM cannot! For instance, if you call SMS_enableSRAM() from code
in bank2, your code gets replaced by the SRAM contents...

## Authors

* **Raphaël Assenat**


## Licence

	Permission to use, copy, modify, and/or distribute this software for any
	purpose with or without fee is hereby granted, provided that the above
	copyright notice and this permission notice appear in all copies.

	THE SOFTWARE IS PROVIDED "AS IS" AND THE AUTHOR DISCLAIMS ALL WARRANTIES
	WITH REGARD TO THIS SOFTWARE INCLUDING ALL IMPLIED WARRANTIES OF
	MERCHANTABILITY AND FITNESS. IN NO EVENT SHALL THE AUTHOR BE LIABLE FOR ANY
	SPECIAL, DIRECT, INDIRECT, OR CONSEQUENTIAL DAMAGES OR ANY DAMAGES
	WHATSOEVER RESULTING FROM LOSS OF USE, DATA OR PROFITS, WHETHER IN AN ACTION
	OF CONTRACT, NEGLIGENCE OR OTHER TORTIOUS ACTION, ARISING OUT OF OR IN
	CONNECTION WITH THE USE OR PERFORMANCE OF THIS SOFTWARE.

Basically, you can use this freely, even for commercial projects, without opening
your code. However, while not required, a mention somewhere if you use this code would be
appreciated. And if you improve something, pull requests would be appreciated too!


