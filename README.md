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

## 2. Soldering
![PIGPIO](https://github.com/Nico-Hei/GNW-Modding/blob/main/RPI_GNW_Pinout.png)
*IMG source: https://pypi.org/project/gnwmanager/*

I own the Zelda Version, as you can see the Pin Layout is the same the only difference is the 4MB available storage on the Zelda version and 1MB on Mario version.
I soldered 2 GPIO cables on the SWDIO and SWCLK ports without any problem but then fried the ground pad on the debug holes so i soldered the ground gpio cable to the usb c port.
### Disconnect the battery while soldering!!!

## 3. Setting up *gnwmanager*
We need gnwmanager because it provides a lot of usefull tools like flashing the chip banks of the gnw as well as unlock its bootloader.
SSH into programmer. 

Then iam installing pipx: `sudo apt install pipx -y` (pipx is like pip but with built in virtual environments(only used for python applications, does not replace pip)).

After this we can run `pipx install gnwmanager` to install gnwmanager.

To add gnwmanager to the pi's path environment variables we have to run `pipx ensurepath` and lock out and in again real quick.

I think openocd is mainly used to unlock the gnw's bootloader but we still have to install it as its required by gnwmanager
`gnwmanager install openocd`

Run `gnwmanager info` if following message pops up you installed it correctly
![GNWInfo](https://github.com/Nico-Hei/GNW-Modding/blob/main/RPI_GNW_Installation.png)
The error is okay. It appears because I havent connected the game and watch to the pi yet.

## 3. Connecting 
Connect your gnw to the pins on your rpi as shown on the image in the soldering step.
Then either connect you battery again or charge the gnw via the usb c port.
**If on any of the next steps you get weird errors check if your screen is on. The gnw has to be on most of the steps.**

Make sure `gpiod is installed`.

As I try to use `gnwmanager info` again the same error as before connecting occours.

Using `GNWMANAGER_VERBOSITY=debug gnwmanager info` i get following error:
![GNWError](https://github.com/Nico-Hei/GNW-Modding/blob/main/GNWError.png)
Openocd couldnt find our gpio pins using what i think is a special "sysfsgpio" config.

Using `ls /usr/share/openocd/scripts/interface/` i can see that the "sysfsgpio-raspberrypi.cfg" exists.
As testet my first try doing this i need to use the "raspberrypi-swd.cfg" file.
This tells me i need to define the interface config myself:
`openocd -f interface/raspberrypi-swd.cfg` but i also need to configure a target according to the output so i use:
`openocd -f interface/raspberrypi-swd.cfg -c "transport select swd"`. This still wasnt able to connect to my gnw.

Through trial and error i found out i needed another arg. `openocd -f interface/raspberrypi-swd.cfg -c "transport select swd" -f target/stm32h7x.cfg`
Please dont ask me what exactly "stm32h7x.cfg" is. Maybe a config file for the gnw's cpu.

After running this command iam finally able to speak to the gnw directly.
![GNWConnected](https://github.com/Nico-Hei/GNW-Modding/blob/main/GNWConnected.png)
As you can see the CPU gets shown correctly "Cortex-M7 r1p1 processor detected" and the connection is ready to be used "Examination succeed".

Now we have to configure this config for gnwmanager.
I found that pipx saved our gnwmanager openocd data in `.local/share/pipx/venvs/gnwmanager/lib/python3.13/site-packages/gnwmanager/ocdbackend/`

In the file `.local/share/pipx/venvs/gnwmanager/lib/python3.13/site-packages/gnwmanager/ocdbackend/openocd_backend.py` I scrolled down a bit finding the different interface settings and changing the raspberry pi's to:

```python
cmd = base_cmd.copy()
cmd.extend(["-c", "adapter speed 1000"])
cmd.extend(["-f", "interface/raspberrypi-swd.cfg"])
cmd.extend(["-c", "transport select swd"])
cmd.extend(["-f", "target/stm32h7x.cfg"])
yield "rpi-gpio", cmd
```
*[File](https://github.com/Nico-Hei/GNW-Modding/blob/main/openocd_backend.py)*

`GNWMANAGER_VERBOSITY=debug gnwmanager info` now outputs correctly

## 4. Backups and bootloader unlock
I already did this step but normally you should use `gnwmanager unlock` now and save the created backup files.

## 5. Retro-Go installation
Now we are going to install retro go as our firmware as it provides us with our game emulators.
Iam not conducting or promoting game piracy. Please try to gather legally optained game copys.

First clone the Retro Go repo with this command to install necessary sub modules
`git clone --recurse-submodules https://github.com/sylverb/game-and-watch-retro-go -b filesystem_wip`
Because of this not used emulators shouldnt be compiled every time. This saves storage and time.
| I needed to install git at first `sudo apt install git`

Then `cd game-and-watch-retro-go` cd into the folder.

Install pip `sudo apt install python3-pip -y`

Install venv `sudo apt install python3-venv -y`

Create virtual environment for project: `python3 -m venv venv`

Enter venv: `source venv/bin/activate`(Deactivate to exit it)

Install requirements: `python3 -m pip install -r requirements.txt`

Install arm-gcc-none-eabi toolchain(V.>=10): `sudo apt install gcc-arm-none-eabi`

## 6. Adding games and retro go flashing
Now add your roms to retro go. The mario version has 1mb of base storage the zelda version 4mb. You will have to inform yourself about compression methods in case of wanting to add multiple games.
[GNWRomFolders](https://github.com/Nico-Hei/GNW-Modding/blob/main/GNWRoms.png)

Stay inside your retro go venv.

! **DO NOT MINDLESSLY RUN** I had to reset my flash banks: `gwnmanager erase all` because of storage errors.

Run `make clean`
Run `make -j4 GNW_TARGET=zelda` (or =mario, depending on your system)

Exit venv `deactivate`

Flash firmware: `gnwmanager flash bank1 build/gw_retro_go_intflash.bin`

Flash games etc.: `gnwmanager flash ext build/gw_retro_go_extflash.bin`

## 7. Possible additions:
1. SD Card Slot and support
2. Larger flash chips (Up to 64mb)
   -> Homebrew support
3. Programming GNW via USB-C Port
4. Cover Art und UI designs
