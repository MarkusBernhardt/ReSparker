# ReSparker
ReSparker is currently a collection of information around the grinding rings for the Sparx skate sharpening machine. With this information, the right hardware and some technical know-how can grinding rings, which have been disabled by the Sparx skate sharpener after 320 grinding cycles, be reset and activated again. In the long run I would like to add an iOS and Android app to automate the process, but this is for another day.

## Why
My main goal of this project is the protection of the interests, rights and freedoms of the owners of a Sparx skate sharpener machine.

I can fully understand the interest of a company to protect its intellectual property, copyrights, patents, whatsoever, and the need to give the user guidance on the lifetime of the grinding rings. But I also think this is not a one way street. The interests and rights of a company end where the interests and rights of their customers start. Where that exact border is, must be defined by the society through the lawmaker and not by companies or customers.

That said. Yes, i am „messing into chip and cycle setup made by Sparx“. But the relevant question for me is, whether this setup by Sparx is lawful. And there I strongly believe this is currently only tolerated (meaning it would be even lawful to reverse engineer the firmware) and will be unlawful from mid 2026 on in the EU.

Car manufacturers had to learn that. They wanted to decide, who is allowed to repair a car, who is allowed to produce parts, who is allowed to sell parts, who has access to the management software, and so on.

Telephone companies had to learn that. When I was young it was an indictable offense to connect a computer modem to a telephone line.

Printer manufacturers had to learn that. They wanted to control, who is allowed to refill consumables, produce generic consumables, and so on.

Sparx also needs to learn that disabling a ring by the skate sharpener machine is a no go. There are a lot more ways to ensure the quality of the grinding. Simply display a big, fat, scary warning. This is also how your car or your printer are doing it. They still work, when you miss the oil change interval of your car or use non-genuine printer ink. They only displays a warning and store that somewhere in the system, so you cannot sue them, when your engine or print head explodes.

## Status
Most of the data written on a grinding ring is decoded by now.

By now there are two problems left:
1. The Sparx skate sharpener calculates from the first 4 bytes of the NFC UID on the grinding rings a password. Without this password only the first 9 (or 11) records of the grinding rings can be accessed. On the grinding ring is an additional copy of the relevant data stored in later records and when the Sparx skate sharpener detects a difference between both copies it erases the ring. So we cannot simply change the data in the unprotected records. I was not able so far to determine the algorithm how the sharpener calculates the password. Luckily there is a way to read the password from the sharpener itself, but this can at the moment only be done with a Flipper Zero. I tried to port the method to iOS and Android, but both systems block some required NFC commands.
2. The Sparx skate sharpener itself stores a copy of the data from the last ?? rings. When the machine detects a difference between the ring and the stored data it again erases the ring. The machine identifies the ring by the Sparx serial of the ring. So, when you want to use a modified ring in the same machine, as it was used before, then you also must change the Sparx serial. But there is also a checksum in there that I do not know how to calculate right now. At the moment I simply overwrite the Sparx serial with the serial of another ring from a friend.

## Data Layout
This is, how far we understand the layout by now: [Data Layout](https://github.com/MarkusBernhardt/ReSparker/blob/main/doc/ring-data-layout.txt)

## Unlocking a ring
Current rings are locked with a password. For old rings is this step not required.

There might be multiple ways to read the password, but this is how I do it:
### Alternative 1: Flipper Zero
* Get a [Flipper Zero](https://flipperzero.one)
* Remove any ring from the Sparx sharpener machine
* Close the Sparx sharpener machine and switch it on
* On the Flipper Zero go to NFC -> Read
* Read the ring you want to unlock
* Press More -> Unlock -> Unlock With Reader
* Press the Flipper Zero at the glass of the Sparx machine, where the NFC ring is.
* Wait for the Flipper Zero to catch the password
* Write it down

### Alternative 2: Proxmark 3
* Get a [Proxmark 3 Easy](https://proxmark.com/proxmark-3-hardware/proxmark-3-easy)
* Mount the ring to unlock into the Sparx sharpener machine
* Keep the Sparx sharpener machine open
* Put a little magnet on the left side of the machine. Look where the little magnet is screwed to the base. Put you magnet at the same spot at the opened upper section.
* Activate sniffing on the Proxmark 3
```
pm3 --> hf 14a sniff
```
* Push the hf antenna of the Proxmark 3 directly on the black NFC reading ring of the Sparx sharpener machine
* Switch the Sparx sharpener machine on and wait until the ring is recognized. The orange LED on the Proxmark 3 should be on/flickering during this time
* Click the button on the Proxmark 3 to stop sniffing
* Show the sniffed data
```
pm3 --> hf 14a list
```
* Search for the password in the data
```
51639632 |   51647792 | Rdr |1B  F8  91  38  6C  A4  63        |  ok | PWD-AUTH: 0xF891386C
```

## Resetting the ring
For all this I am using a Proxmark3 V3 Easy NFC kit.

### Dump
Check that everything works with a simple dump. Rings that require a password will only show the first records. Old rings all 45 rows.
```
pm3 --> hf mfu dump
```

If a password is needed, get it with a Flipper Zero and add it.
```
pm3 --> hf mfu dump -k DEADBEEF
```

### Reset counters
Reset the counter to 320 cycles.
```
pm3 --> hf mfu wrbl -k DEADBEEF -b 06 -d 40014001
pm3 --> hf mfu wrbl -k DEADBEEF -b 10 -d 40014001
```

On my ES200 machine there is a maximum count of 4,095 cycles. If you feel adventurous you can try the following and never need to reset the counter again.
```
pm3 --> hf mfu wrbl -k DEADBEEF -b 06 -d FF0FFF0F
pm3 --> hf mfu wrbl -k DEADBEEF -b 10 -d FF0FFF0F
```

### Clean up 
If there is any data in rows 16 to 33 (not 00000000), clean it.
```
pm3 --> hf mfu wrbl -k DEADBEEF -b 16 -d 00000000
pm3 --> hf mfu wrbl -k DEADBEEF -b 17 -d 00000000
...
pm3 --> hf mfu wrbl -k DEADBEEF -b 33 -d 00000000
```

### Change the Sparx serial
This is needed, when changing a ring that should be used in the same Sparx machine again. The serial is stored in the first two bytes of record 7 and all bytes of record 8. The copy of the serial is stored in the first two bytes of record 11 and all bytes of record 12. Records 8 and 12 can be simply overwritten. Rows 7 and 11 must be read, the first 2 bytes changed and written back.
```
pm3 --> hf mfu rdbl -k DEADBEEF -b 7
pm3 --> hf mfu rdbl -k DEADBEEF -b 11

pm3 --> hf mfu wrbl -k DEADBEEF -b 7 -d 0000xxxx
pm3 --> hf mfu wrbl -k DEADBEEF -b 8 -d 00000000
pm3 --> hf mfu wrbl -k DEADBEEF -b 11 -d 0000xxxx
pm3 --> hf mfu wrbl -k DEADBEEF -b 12 -d 00000000
```

# Change ring type
The machine type is encoded in the byte 4 of record 7 and 9. Commercial machines are defined as 14. Consumer machines use 6. The hollow of the ring is encoded in byte 3 of record 7 and coded differently for consumer and commercial rings. See the list in [Data Layout](https://github.com/MarkusBernhardt/ReSparker/blob/main/doc/ring-data-layout.txt). When you encounter a ring with an hollow not marked as confirmed, please give me a short notice.
```
pm3 --> hf mfu rdbl -k DEADBEEF -b 7
pm3 --> hf mfu rdbl -k DEADBEEF -b 9

pm3 --> hf mfu wrbl -k DEADBEEF -b 7 -d xxxx2506
pm3 --> hf mfu wrbl -k DEADBEEF -b 9 -d xxxxxx06
```

## License
All files in this repository are covered by one of two licenses:
* [Creative Commons Attribution Non Commercial Share Alike 4.0 International](https://github.com/MarkusBernhardt/ReSparker/blob/main/LICENSE-CC-BY-NC-SA-4.0.txt)
* [GNU Affero General Public License 3.0](https://github.com/MarkusBernhardt/ReSparker/blob/main/LICENSE-AGPL-3.0.txt)

The default license throughout the repository is CC BY-NC-SA 4.0 unless the file header specifies another license.

