
# MKS THR 36 CAN connection to MKS SKIPR board 

Each board needs two compilation and flashing processes - CanBoot and klipper.
* CanBoot is a bootloader that helps us flash, klipper is flashed through it.
* Klipper is Klipper


Mellow has very good guides for their toolheads, like this one (rp2040 just like a THR36) : https://mellow-3d.github.io/fly-sb2040_canboot_can.html 

Similar MCU from makerbase with CAN https://github.com/maz0r/klipper_canbus/blob/main/controller/monster8v2.md



Quick summary of steps: 

* 1.1 Make CanBoot for MCU
* 1.2 Flash CanBoot to MCU with USB-DFU (long press boot)
* 1.3 Prepare Klipper firmware bin file for MCU 
* 2.1 Prepare Canboot bootloader for THR
* 2.2 Flash CanBoot to THR with USB (long press boot)
* 2.3 Prepare Klipper firmware bin file for THR
* 3.1 Flash Klipper to THR with USB-CanBoot (double reset)
* 3.2 Flash Klipper to MCU with USB-CanBoot (double reset)
* 4.1 Run CanBoot script to scan CAN and save UUIDs


## Instructions
* Klipper version must be at least v0.11.0-194-g1a24e7c5!
* Connect host part of SKIPR to MCU part with USB (2.0 in my case enough)
 
![image](https://github.com/TrueFargo/Ai3MS-Skipr/assets/115958663/e1a0dc8e-f1d4-4a7d-966d-2cd78cfa99d2)

* Connect host Skipr to THR through USB and CAN (jumper USB power - off)  (*USB cable for the toolhead is only needed for flashing the firmware*)

![image](https://github.com/TrueFargo/Ai3MS-Skipr/assets/115958663/7bad5296-12a6-43c9-967c-01e1cf2cf306)

###### *After several attempts, I still could not flash the toolhead through CAN, even when it worked properly in klipper firmware with only a CAN connection.*
* Hold boot button on THR before power on

![image](https://github.com/TrueFargo/Ai3MS-Skipr/assets/115958663/dcc1cf09-9230-4e22-ba57-a8fdb28237f2)

### Login to PI part of SKIPR

DISABLE KLIPPER SERVICE  

	sudo service klipper stop

Create or setup Can0 like this:

```
sudo nano /etc/network/interfaces.d/can0
```
```	
auto can0
allow-hotplug can0
iface can0 can static
 bitrate 500000
 up ifconfig $IFACE txqueuelen 1024
```


	
install CanBoot

	cd ~
	git clone https://github.com/Arksine/CanBoot

Install pyserial

	pip3 install pyserial 
or

	pip install pyserial 
	
	


#### Note:
There is a problem here, since I initially used the Monster8 guide - I flashed the MCU with 32KB offset, which, I think, broke the bootloader that loads the firmware from the flash card, since the default settings recommended by the manufacturer are 48KB offset (so even in the klipper settings specified).

CanBoot at the time when I was flashing it did not have a 48 KB offset option at all. I tried manually setting the address in DFU mode and flashing the klipper, but nothing worked for me.

However, **the DFU-mode of the board works fine (long press the "boot" button), and the flashed canboot bootloader also works fine (double-click the reset button)**.
If, for some reason, you are disappointed in the toolheads, and decide to return everything as it was, then just compile klipper for USART1, flash the MCU as well, it will be visible by 
	
	serial: /dev/ttyS0
	
	
	
Makerbase has not yet posted the default bootloader for SKIPR, and I decided not to try the monster8 bootloader.
But maybe all this is nonsense, the default bootloader is written somewhere else, and I just don’t have a suitable flash card (none of the 3 different ones with different file systems fit).

# Flash at your own risk!

	
## SKIPR MCU prepare
### Prepare CanBoot bootloader for MCU
		
	cd ~/CanBoot
	make menuconfig
	
![image](https://github.com/TrueFargo/Ai3MS-Skipr/assets/115958663/a3d06e82-8513-4c47-b317-d3880491ca69)

 	make clean
	make
      
Switch SKIPR MCU to DFU mode for flashing (hold boot0 button on mcu part)

![image](https://github.com/TrueFargo/Ai3MS-Skipr/assets/115958663/fe5d2467-96dc-43b5-b466-8eff07270983)

Check usb

```
lsusb 
```

![image](https://github.com/TrueFargo/Ai3MS-Skipr/assets/115958663/e9e9b619-52a2-43b5-ab20-d3de861abe45)

And now MCU part of the SKIPR board connected via USB is displayed in the DFU mode
#### Flash Canboot to MCU SKIPR
```
sudo dfu-util -a 0 -D ~/CanBoot/out/canboot.bin --dfuse-address 0x08000000:force:mass-erase:leave -d 0483:df11
```
![image](https://github.com/TrueFargo/Ai3MS-Skipr/assets/115958663/a8fc37f9-4f31-4781-b3fd-47e918f43327)


*At this point in the installation, there is no klipper firmware on SKIPR, only a CanBoot bootloader, and in this case, I have a canbus UUID displayed on every request.*

			
### Prepare Klipper for MCU
			
	cd ~/klipper
	make menuconfig
	
![image](https://github.com/TrueFargo/Ai3MS-Skipr/assets/115958663/83426c18-1f5f-4cc2-b921-a0435596ddfa)	
setup f407 32kb USB-to-Can (USB on PA11,PA12); CAN bus on PB12,PB13; 500k speed, 
	
	make clean
	make
compile, rename klipper.bin file and move somewhere *(for example, /home/mks/fw/klipperMCUCAN32k500k.bin)*
			
## THR toolhead prepare
			
### Prepare Canboot bootloader for THR

Hold boot button on THR before power on SKIPR *(If you didn't do it at the beginning, or reset it since then)*
![image](https://github.com/TrueFargo/Ai3MS-Skipr/assets/115958663/dcc1cf09-9230-4e22-ba57-a8fdb28237f2)


	lsusb
	

![image](https://github.com/TrueFargo/Ai3MS-Skipr/assets/115958663/1cab9063-92b9-4b93-8024-1f0ba07b9d1f)

```
cd ~/CanBoot
make menuconfig
```

![image](https://github.com/TrueFargo/Ai3MS-Skipr/assets/115958663/e86ae91d-7f0f-4674-ad8b-5d2f0cb47cd6)

setup rp2040; flash chip w25q080;no deployment; usb comunication(for bootloader) compile and flash trough boot (hold boot before powering on)

```
make clean
make
```
### Flash Canboot bootloader for THR

```
sudo make flash FLASH_DEVICE=2e8a:0003
```	
![image](https://github.com/TrueFargo/Ai3MS-Skipr/assets/115958663/8eef8adb-fbdd-43f7-8d5f-c1dcb138a4ff)


now with double reset on both board we can see 2 device in Canboot mode
		
	~/klipper$ ls /dev/serial/by-id/*
	/dev/serial/by-id/usb-CanBoot_rp2040_35CE4D1503954858-if00  /dev/serial/by-id/usb-CanBoot_stm32f407xx_1D0038001250335330373220-if00
	
### Prepare Klipper for THR

	cd ~/klipper
	make menuconfig

![image](https://github.com/TrueFargo/Ai3MS-Skipr/assets/115958663/67c5f63c-0fb3-48de-b955-a2d0e0aded88)

setup rp2040; 16kb; offset; ,CAN on 8,9; 500k speed; 

	make clean
	make
			
compile; rename klipper.bin file and move
			
			
### Flash Klipper to THR,
Here is what you should see:

	
```	
mks@mkspi:~/klipper$ python3 ~/CanBoot/scripts/flash_can.py -d  /dev/serial/by-id/usb-CanBoot_rp2040_35CE4D1503954858-if00 -f ~/fw/klipperTHRCAN16500.bin
Attempting to connect to bootloader
CanBoot Connected
Protocol Version: 1.0.0
Block Size: 64 bytes
Application Start: 0x10004000
MCU type: rp2040
Flashing '/home/mks/fw/klipperTHRCAN16500.bin'...

[##################################################]

Write complete: 114 pages
Verifying (block count = 456)...

[##################################################]

Verification Complete: SHA = F1E1C2F4CFF37C2C0FEF6AD5A32C80419D477710
CAN Flash Success
```			
### Flash Klipper to MCU SKIPR

	~/klipper$ python3 ~/CanBoot/scripts/flash_can.py -d  /dev/serial/by-id/usb-CanBoot_stm32f407xx_1D0038001250335330373220-if00 -f ~/fw/klipperMCUCAN32k500k.bin

output:
```
Attempting to connect to bootloader
CanBoot Connected
Protocol Version: 1.0.0
Block Size: 64 bytes
Application Start: 0x8008000
MCU type: stm32f407xx
Flashing '/home/mks/fw/klipperMCUCAN32k500k.bin'...

[##################################################]

Write complete: 2 pages
Verifying (block count = 473)...

[##################################################]

Verification Complete: SHA = DD9C0FFDA8AA60754BEB82DC665919F7F0C84B7E
CAN Flash Success
```

### Run CanBoot script to scan CAN
```
~/klipper$ ~/klippy-env/bin/python ~/klipper/scripts/canbus_query.py can0
```
	
output:

	Found canbus_uuid=92feb5cf8e71, Application: Klipper
	Found canbus_uuid=3662e33915a1, Application: Klipper
	Total 2 uuids found

#### Save these IDs as soon as you see them for the first time! 
##### They do not change, but after the first request, subsequent querys give out "0 uuid found" and,in my case, CAN bus drops and does not turn on until the board is physically turned on / off or a reset is pressed on the MCU board (FIRMWARE_RESTART not working with this). Despite this, the Klipper sees and controls all boards absolutely normally.

Klipper should now see the boards and you can use uuid as the address in printer.cfg 

	
	[mcu]
	# The hardware use USART1 PA10/PA9 connect to RK3328
	# serial: /dev/serial/by-id/usb-Klipper_stm32f407xx_4D0045001850314335393520-if00
	# serial: /dev/ttyS0           	#default usart
	canbus_uuid: 92feb5cf8e71      	#after canboot flashing and klipper through canboot flashing
	;restart_method: rpi_usb 		#for can bus not needed
	
	[mcu rpi]       
	serial: /tmp/klipper_host_mcu
	
	[mcu MKS_THR]
	# serial: /dev/serial/by-id/usb-Klipper_rp2040_35CE4D1503954858-if00 ##usb id
	canbus_uuid: 3662e33915a1
	
	
	
## Issues

### Which TMC? 2209 or 2208?
I setup 2209 as usual in uart mode

### THR show shows the temperature is 5 degrees less than it should 
*(Easy to check when the printer is idle for a long time - everything should be at room temperature - both the bed and the nozzle)*
Here config with generic b3950 thermistor:
```

## custom table for THR thermistor from MKS with with a little tweak & extruder part of cfg
[adc_temperature extruder]
temperature1:27
voltage1:3.151
temperature2:34.8
voltage2:3.10
temperature3:42
voltage3:3.01
temperature4:81.0
voltage4:2.354
temperature5:143
voltage5:1.029
temperature6:171
voltage6:0.632
temperature7:195
voltage7:0.401
temperature8:225
voltage8:0.250
temperature9:255
voltage9:0.159
temperature10:287
voltage10:0.098


[extruder]                          ## SHERPA mini
# step_pin: PB5			    #SKIPR
# dir_pin: PB4                      #SKIPR
# enable_pin: !PB6                  #SKIPR
step_pin: MKS_THR:gpio5             # thr36
dir_pin: MKS_THR:gpio4              # thr36
enable_pin: !MKS_THR:gpio10         # thr36
microsteps: 16                      
gear_ratio: 50:10 					#for standard 10t motor
full_steps_per_rotation: 200        #1.8deg Motor
rotation_distance: 22.67895         #for 5mm Shaft Driven Bondtech gearsets

nozzle_diameter: 0.400
filament_diameter: 1.750
pressure_advance = 0.041200   		; do your measure
pressure_advance_smooth_time: 0.001	; do your measure


# heater_pin: PB1                   # HE0 skipr
# sensor_pin:PC2                    # THM Interface th0
# sensor_type: Generic 3950       

heater_pin: MKS_THR:gpio0           # thr36
sensor_pin: MKS_THR:gpio26          # thr36

sensor_type: extruder           			# thr36 thermistor corrections
adc_voltage: 3.3					# thr36 thermistor corrections
voltage_offset: 0.00					# thr36 thermistor corrections
#   The ADC voltage offset (in Volts). The default is 0.
#pullup_resistor:4700
#inline_resistor:0


max_power: 1.0
min_temp: 0
max_temp: 290
max_extrude_cross_section: 4 
max_extrude_only_distance: 1400.0
max_extrude_only_velocity: 75.0
max_extrude_only_accel: 1500
```
	
	
	
Other configs and pinouts can be found on the MKS github
	
