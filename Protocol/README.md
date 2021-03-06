# How I can help? What can I do?

If you have cripto-analysis skill, programming skill, or hacking skill (like using a disassembler/debugger) you can help me understanding and decoding how [VigoWork and ESP32 exchange information in encripted way](https://github.com/arkypita/Vigotech-VG-L7X/tree/main/Protocol#vigo-encripted-messages).

You can use a serial logger, like [eltima serial logger](www.eltima.com/products/rs232-data-logger) or any other freeware, to dump messages between VigoWork and ESP2.
You also can use a disassembler/debugger like [IDA/IDA Free](https://www.hex-rays.com/products/ida/) or [PE Explorer](http://www.heaventools.com/PE_Explorer_disassembler.htm)
to follow disassembly VigoWork executable code and try to reverse-engineering the cripting routine.

If you want you can [build your-own fake VG-L7x board](https://github.com/arkypita/Vigotech-VG-L7X/blob/main/Hardware/README.md#buid-your-own-fake-vg-l7x-controller-to-test-the-protocol) for testing, by flashing the original Firmware on a bulk ESP32 proto board, wired to Arduino Nano/UNO board with standard GRBL firmware.

## Standard GRBL protocol

GRBL protocol is public and well [documented](https://github.com/gnea/grbl/wiki/Grbl-v1.1-Interface).

Essentially PC software (like LaserGRBL) send a gcode command i.e. **G0 X100 Y100** followed by **\n** newline character, the board receive the command and reply with an **ok\n** message or with an **error:x\n** message. The reception of the reply tell the sender that the command has been handled and then next commands of a job can be sent.

Also, the program may periodically send a "**?**" which asks the machine to send a status message containing [real-time information](https://github.com/gnea/grbl/wiki/Grbl-v1.1-Interface#real-time-status-reports) such as current position, alarm/idle/run status and other data.

That's all you need to know.



## VigoWork protocol (how VigoWork and L7x board talk)

**NOTE:** following info were deduced by intercepting and spying the USB communication between VigoWork software and its board using [eltima serial logger](www.eltima.com/products/rs232-data-logger).

The first difference is that Vigo does not use "**?**" but use character "**0x88**" to query status, but this is not a big issue.

Problems start with the difference in how Vigo stream the command, that is, in the way in which the machine confirms receipt of commands.

Essentially the gcode commands are sent "as-is" in the same way as grbl does, but no "**ok**" and no "**error**" messages are sent back from the board. Instead of ok/error response a special status message is sent from the board. This status message has this format:

`<VSta:2|SBuf:5,1,0|LTC:4095>`

Numbers after SBuf: count how many "ok" and how many "error" the ESP32 "see" by talking with grbl/Atmega328.
PC software can use that numbers by translating them to the missing "ok" and "error" messages.

The major issue is that this status message is not enable "by default", but should be activated by send special **encripted** commands to Vigo board.

## Vigo encripted messages

**1. General concepts**
- All "scrambled" messages that VigoWork and its board exchange begin with the ">" character
- the exchange of coded messages can take place either on the initiative of the board (the board sends a message, VigoWork responds) or on the initiative of the VigoWork program (VigoWork sends a message and the board replies).
- With the same function (connection, start of stream sending, end of stream sending) the messages exchanged are always different, but keep the same length in characters.
- If you send a previously captured message, the board receives it validly, and responds as expected.
- if you make some small changes to a valid message (i.e. swap some characters) the board ignores it and does not answer anything.
- Vigo protocol mix "plain text" human readable command stream (like gcode and status report) and encoded messages.

Here is an example of exchange of messages

![example](https://user-images.githubusercontent.com/8782035/95726102-ca9fa100-0c78-11eb-9425-2039875e311c.png)

**2. Encripted messages at connect**

Each time VigoWork open a connetction to VG-L7x board, VigoWork sends a message like this:

`>Qvq9fceN7KBak5hPlOBRzgzhvexBHSORPyMOVC:wKiUiai`

VG-L7x board reply with a message like this:

`>OPXci7m9I6qeAdz7tdcJisOO59GcMcsM3Oh:GA`

You can see some of this messages I captured in file ["connect.txt"](https://github.com/arkypita/Vigotech-VG-L7X/blob/main/Protocol/connect.txt) contained in this folder.


**3. Encripted messages at board power on**

If VigoWork is connected to the board, and the board is switched off then switched on, at first messages received VigoWork send a message to the board that look like this:

`>Qvref7f8IqBrFKgPDqEaO5badamayK4blG`

VG-L7x board reply with a very long message like this:

`>P:R9VeBsK6n99RzAxxjcMS/CQqHKUTmCQKrZ6abDlOBduct8gJfMP92OL7PMsM69Wz7MNMErq5sKL6L7IKF65qraHr85CKHbM7XqYjD7N7FrnKsqFbJcEKEL4apKIr7q7KLLRMYaUT4ahPtS`

Captured messages of this type on file ["poweron.txt"](https://github.com/arkypita/Vigotech-VG-L7X/blob/main/Protocol/poweron.txt) contained in this folder.

**4. When VigoWork start streaming a Job**

This message is really very important, in fact it activates a board mode that enable the board sending the "buffer-status-report", which is indispensable to write a streaming algorithm that does not fill the reception buffer.

VigoWork send: 

`>O:m9jMd8L5fK75df09Ly7hoxXiOebiCgKCWy3E`

VG-L7x board reply:

`>Q:Qck8UNJ6OOUeC7IOXuvdRNg:8l`

Captured messages of this type on file ["beginsend.txt"](https://github.com/arkypita/Vigotech-VG-L7X/blob/main/Protocol/beginsend.txt) contained in this folder.

**5. When VigoWork finish streaming a Job**

This message is also important, warns the board that the last message has been sent and that no more will arrive for this Job.
VigoWork sends it at the end of the stream, after the last gcode command and after a "return home" command (G0 X0 Y0 M5).

VigoWork send: 

`>TflOecd8JKd5FqcP1eGiAhrBWii7rBXAjZ`

VG-L7x board reply:

`>O:O7q8adELicmNXMydMNd::D`

Captured messages of this type on file ["endsend.txt"](https://github.com/arkypita/Vigotech-VG-L7X/blob/main/Protocol/endsend.txt) contained in this folder.
