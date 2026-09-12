# GNW-Modding

I decided to mod my Zelda Nintendo Game and Watch, why?
I wanted to play pokemon games on the go relying on the GNW's massive 6-8 hours of battery life 
as well as its quick resume ability.
I have already modded the GNW yesterday by using ai and 8 hours of my time. Because of some bugs which were maybe caused by
copying and pasting every command the ai printed out mindlessly I am going 
to redo the whole process of flashing the firmware as well as emulator and document all of it.

### PLEASE do not use this repo as a guide to modding a GNW yourself. I have allready messed up my device in the first try by bricking the custom firmware
### before being able to back it up (Unlocking the flash deleted everything).

## 1. Setting up the programmer
~~Iam using a **Raspberry PI Zero W**(the old version). With its one core its not the fastet machine to build c but with enough time it is working good enough.
Ive set it up with base 32 bit raspberry pi os lite and activated ssh.~~

My RPI0 is no longer in use and is being replaced by my RPI4(8gb). The RPI0 crashed to many times in the process on me while installing packages and was overall because of its speed not fun to use.
The RPI4 might be a bit overkill but should help in the process of building packages later on.
I installed raspberry pi os lite (64bit), set up wifi and ssh via password as this pi is not going to run any important services.

First lets update the pi
`sudo apt update && apt upgrade -y`

Next I check for the installed python. Regarding the gnwmanager installation guide we need >=3.9
`python --version` shows me that I currently have version 3.13.5 pre installed.

## 2. Soldering and connecting wires
![PI0GPIO](https://github.com/Nico-Hei/GNW-Modding/blob/main/RPI_GNW_Pinout.png)
*IMG source: https://pypi.org/project/gnwmanager/*

I own the Zelda Version, as you can see the Pin Layout is the same the only difference is the 4MB available storage on the Zelda version and 1MB on Mario version.
I soldered 2 GPIO cables on the SWDIO and SWCLK ports without any problem but then fried the ground pad on the debug holes so i soldered the ground gpio cable to the usb c port.
### Disconnect the battery while soldering!!!
Then just connect the gpio cables to the relating gpio pins on the pi.

## 3. Setting up *gnwmanager*
We need gnwmanager because it provides a lot of usefull tools like flashing the chip banks of the gnw as well as unlock its bootloader.
Iam not going to cover the bootloader step as ive allready done it on my first try. **Important side note** be carefull! While unlocking my bootloader
my base firmware got deletet because of nintendos security measurments. I was not able to backup this firmware unfortunatly.
SSH into programmer. 

Then iam installing pipx: `sudo apt install pipx -y` (pipx is like pip but with built in virtual environments).

After this we can run `pipx install gnwmanager` to install gnwmanager.

To add gnwmanager to the pi's path environment variables we have to run `pipx ensurepath` and lock out and in again real quick.

I think openocd is mainly used to unlock the gnw's bootloader but we still have to install it as its required by gnwmanager
`gnwmanager install openocd`
