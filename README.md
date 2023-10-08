Mini-WSPR
WSPR Beacon implementation with ATmega328P and 328eForth programming system

Having an interest in the programming language Forth I came across the work of C. H. Ting and the Silicon Valley Forth Interest Group (SVFIG) where a system of Forth for the Atmega328P microprocessor was created. Since Forth is an Interpreter and Compiler in the same system, code can be developed on the native system (Atmega328P) tested and debugged and then executed to produce a WSPR beacon. The basic 328eForth system developed by Dr. Ting uses only 5.6K of the 32K flash memory in the Atmega328, leaving enough room for adding code for a basic amateur radio project with an SI5351A programmable VFO. The Forth system has access to all the 328’s registers, interrupts and memory which makes low level programming feasible without going to external compilers or assemblers.

The first task was to implement code “words” for I2C communication from the 328 to the SI5351 and other peripheral devices (1306 OLED display, 25264 EEPROM and DS 3231 Real Time Clock Module). No libraries of code are used as with the Arduino C++ development environment, only examples from other programmers using forth or C. Here the development environment (IDE) is the system!

The second task was to develop a series of code routines (words) to do utility tasks and Double word (32 bit) and Quad word (64 bit) integer math operations. The basic system used 16 bit words and 8 bit bytes. All calculations use Integer math and 64 bit operations are needed for some of the calculations (i.e. divide 64 bit number by 32 bit number for 32 bit quotient and 32 bit remainder). Nothing is unique about the math routines created, it was just the first thing I tried that worked correctly.

Programming the 5351

Several sources were used to help with the code for the SI5351 with Hans Summer and RFzero Si5351A being noteworthy. Again, I must start at first principles with low level programming to load and change the parameters (registers) on the SI5351. The basic equations for generating frequency output from the SI5351 are.

Fout = (25MHz*(a+b/c))/(d+e/f)	Where 25MHz is the actual clock frequency

The parameters (a + b/c) and (d + e/f) can be calculated based on some simplification assumptions developed by Hans Summers for the SI5351.

The Silicon Labs App Note 619 with definitions of the registers and calculations is some help but leaves much to be desired as a “how to” document.

WSPR (Weak Signal Propagation Reporting)

Ok, now we are going to use an SI5351 to output an RF signal of a lowly 20-30 mW and see what happens. Ah, but George Steber WB9LVI has done a similar project using an Arduino Nano and a SI5351 that was published in QST January 2022 edition. So, it can work with a minimal hardware configuration, and now we can see it will work at the lowest level without C or assembly.

The best guide to the algorithms for generating WSPR signals was the note by Andy Talbot G4JNT in 2009 (see appendix) that walks through the steps of producing a Symbol table to use for the FSK RF signal. This was the hardest section to develop with low level coding as some of the intermediate results could not be validated with examples to help with the next step in the process.

The Input Code string is a 6 character call sign left filled with blanks (hex 20), a 4 char location and 2 digit power dBm. Translate this to a 11 byte binary array then generate a 162 byte FEC Forward Error Corrected array of integers symbols. Next these 162 bytes are interleaved to spread out the symbols from front to back. The same 162 bytes are used for both operations with the low nibble used for the FEC data, and the high nibble used for the Interleaved results. The final symbol table combines the interleaved data with a single bit list of well corelated data to produce a 162 byte array of symbols (bit*2+interleaved) from 0 to 3. Again, we use the same 162 byte array as before and process with 16 bit words representing the 162 bits (10 plus one short) to save space.

This symbol array is used to output the FSK WSPR signal where each digit represents a shift of 1.464 Hz from the base frequency in the WSPR band. For 20M band frequency is 14,095,600 Hz plus 1,400 Hz offset plus WSPR offset, say 14,097,050. Each frequency is high for 683 ms for each of the 162 symbols then delay of 9400 ms for a full 2 minute transmission. Transmissions must start on an even 2 minute time.

After many false starts I found that SI5351 output could be checked by using a second computer running SDR# software and an RTL-SDR dongle attached. First calibrate the SDR# software with WWV to the nearest Hz. When the SI5351 is initialized, it usually will output it’s clock frequency of 25,000,000 Hz +- offset. Determine the clock actual frequency by tuning the SDR# to 25MHz and finding the clock frequency as accurately as possible. For me this was 25,007,325 Hz.

Input the WSPR frequency (base + 1,400) and the calibrated clock (feq> word and clk> word)
Calculate parameters (cal_abcd and cal_all) then set the registers in the 5351 (set_regs). The send_msg routine will output a 2 minute WSPR message at the above frequency settings. The 1.464 Hz frequency shift is produced by changing the 5, 6 and 7 registers (hex address 20,21,22) i.e., parameter b. The sym_set word creates an array of 3 by 4 for each of the 3 registers and the 4 values they need for each frequency step.

For version 6.12, all timing is done with delay loops on the Atmega328P microprocessor. The send_msg word is started manually by watching for a time standard (Dimension 4) clock to turn over on the next even 2 minute interval. A test run of 5 hours shows no loss of timing for WSPR by just using delay loops. The next version 6.22 of the project is to integrate a DS3231 Real Time Clock module for accurate and automatic running of the WSPR transmission. The beacon is started from the band words and waits for a even 2 minute DS3231 time signal

Code words for a .96” 1306 OLED display have been tested and plans are to have a time and frequency display for transmissions. The Atmega328P can be used naked on a development board with only a 16 MHz crystal some 22pF caps a USB to serial module (for the RealTerm serial terminal) and the SI5351A module and a low pass filter.

Currently the hardware is an Arduino UNO R3, a AVR ISP programmer using the AVR Studio 7 software. The 328eForth is compiled by the AVR studio from Dr. Ting’s assembler source code and the Hex file loaded via the ISP interface into the Atmega328P. Because the 328eForth’s requirement to write to Flash the boot loader is overwritten so the ISP is used to load the base system. The system is now active and needs only a serial terminal program to talk to the 328eForth system. The terminal program is then used to load the WSPR-Mini code as a text file which is compiled by the Forth system into executable words. The terminal program must have a 500 ms delay at the end of each line of text transmitted to the 328eForth compiler for the code word creation. The program RealTerm 2.0.0.70 is a free download and works well once configured. For the MiniWSPR version 6.22 the UNO R3 was replaced by a NANO clone and the DS3231 real time clock module added.

This Project does demonstrate that using Forth as a programming language for an Atmega328P microprocessor can create a WSPR beacon without the use of high level programming tools. Forth is ideal in that it is both a compiler and interpreter in the same system as no other development environment is required. You build up your project from simple “words” that can be tested interactively, and then used in the next level of code words then repeat the process until there is a finished project such as the WSPR-Mini described above.

Testing The System

A separate computer with an RTL-SRD dongle attached was used to test the Mini-WSPR system. This second system in the next room used the WSJT-X software to decode any received RF. With the second system the WSPR frequencies and symbol coding were validated. The small oscilloscope was used to check the rough frequency output from the SI5351. Delay loops were adjusted to have 162 symbols of 683 ms followed by a 9400 ms delay time to make a 120 second total transmission time. During a band change the delay time needs to adjusted to take into account the time to redo all the parameter calculations (~70 ms ). The first demo system is shown in the photo below where the clk0 output of the 5351 is connected to a 20M LPF and an EFHW wire antenna. The laptop computer is used as an ASCII terminal to talk to the ATmega328P via the UNO’s USB to TTL ASCII connection.

The WSPR map of a test run of 5 hours overnight demonstrates how a 20 milliwate WSPR beacon is recieved from our QTH in Texas. 

 
![image](https://github.com/rgeorge-228/Mini-WSPR/assets/51888591/3537c8dc-2a6c-4348-9036-64a7094e1683)



![Screenshot 2023-06-01 093527](https://github.com/rgeorge-228/Mini-WSPR/assets/51888591/b126f7f4-d329-4149-94be-d63cc17b51d2)






Robert George		KI5HBO	rgeorge.228@gmail.com		May, 2023 

