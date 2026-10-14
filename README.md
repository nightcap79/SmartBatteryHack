# SmartBatteryHack
This is a hacking tool for smart batteries using SMBus. Originally written for a Dell J1KND battery that uses a [BQ8050](Datasheets/BQ8050_datasheet.pdf) fuel gauge IC.

**Arduino** folder contains the source code for an Uno/Mega which acts as an interface between the battery and an external computer. The I2C pins are connected to the smart battery's SDA/SCL pins with pullup resistors (4.7kOhm to 5V). The battery's ground pin has to be connected to the Uno/Mega's ground pin. Additionally the battery's "system present" pin may need to be grounded too (through a 100-1000 Ohm resistor). You need to install the [SoftI2CMaster](https://github.com/felias-fogg/SoftI2CMaster) library to be able to use this sketch.

**GUI** folder contains the C# source code and compiled binary for an external computer (Windows 7 and up) written in Visual Studio 2019.

Check the [Wiki](https://github.com/laszlodaniel/SmartBatteryHack/wiki) for tutorials.

![GUI 01](https://boundaryconditionhome.files.wordpress.com/2020/01/sbhack_gui_01.png)

![GUI 02](https://boundaryconditionhome.files.wordpress.com/2020/01/sbhack_gui_02.png)

![Battery connection](https://boundaryconditionhome.files.wordpress.com/2020/02/img_20200202_092224_02.jpg)

More info about this project:  
https://boundarycondition.home.blog/2020/01/18/the-repairing-and-hacking-of-a-dell-j1knd-bq8050-laptop-battery/


For Toshiba PA3817U-1BRS:

info:
battery TOSHIBA (PA3817U-1BRS)
PABAS228
10.8V
48Wh
4200mAh
E85792
94V-0
CK 77-1
BQ8050  chipset 07KG4 C18Q


Connections:
Connect pin 7 to 4.7kohm to arduino SCL to 4.7kohm pull-up resistor  to  3.3 v
Connect pin 6 to arduino SDA to 4.7kohm pull-up resistor  to  3.3 v
Connect pin 3 to 100ohm to arduino GND
Connect pin 1 to 4.7kohm to arduino GND



according to Man-Eating Monkey, we need to overcome a challange to unseal the controller

Man-Eating MonkeyApril 8, 2023 at 4:59 AM
Hi, I'd like to share my success hacking a DELL/SANYO T54FJ with BQ8050. I can confirm that following set of very similar steps are needed to unlock the chip.

1. Send 0x0214 to 0x71
2. Read static word from 0x74
3. Read challenge word from 0x73
4. Send ???? to 0x71
5. Send 0x0517 to 0x70

The difference is that step 4 is not a simple subtraction of X from 0x10000, but rather a dynamic value that I believe depends on the challenge that can be read from 0x73, which appears to be a timestamp that increments every 219ms. The good news is that the existence of this challenge does not change the expected time taken to brute force your way in.

Steps 1 and 2 only need to be performed once, but I believe step 3 needs to be done before each attempt of step 4. So in a loop, do a read of 0x73, followed by a write of a random value to 0x71, until 0x70 becomes available. You can copy and paste the following script into PowerShell and it will do the whole process.

With regards to probability, each guess has a 1/65536 chance of succeeding, and you can get through at least 5 attempts a second. This puts the theoretical average time taken at 1.82 hrs. YMMV.

Once you are in, read the EEPROM and zero out the bytes at 0x50C and 0x60C of value A5. This completely bypasses the need for steps 2-4, and in future you can re-enter the boot ROM with steps 1 and 5 only. (This hack also works on BQ8030, although it is less useful). Then zero out the words at 0x580-0x581 and 0x640-0x641 (valued 00 80 for me) to clear the permanent failure. Reflash the EEPROM and run ./smbusb_bq8030flasher --execute to exit out of boot ROM mode.

Reply
Replies

Man-Eating MonkeyApril 8, 2023 at 5:00 AM
This comment has been removed by the author.


Man-Eating MonkeyApril 8, 2023 at 5:03 AM
$write_71 = .\smbusb_comm.exe -a 16 -c 71 -w 0214
if ($write_71) {
echo "Error: battery not connected or already in boot ROM"
} else {
$start = Get-Date
$read_74 = .\smbusb_comm.exe -a 16 -c 74 -r 2
$i = 0
while ($true) {
$hex = '{0:x4}' -f $i
$read_73 = .\smbusb_comm.exe -a 16 -c 73 -r 2
$write_71 = .\smbusb_comm.exe -a 16 -c 71 -w $hex
if ($write_71) {
echo "Lost communication with chip"
break
}
$write_70 = .\smbusb_comm.exe -a 16 -c 70 -w 0517
echo "challenge: $read_73, guess: $hex"
if (!$write_70) {
echo "WE'RE IN!!!!!!!!!!!!!!!!!!!!!!"
break
}
$i = Get-Random -Minimum 0 -Maximum 0xFFFF
}
$end = Get-Date
echo "Started: $start"
echo "Finished: $end"
}
