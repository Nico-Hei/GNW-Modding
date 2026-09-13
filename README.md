# GNW-Modding

I decided to mod my Zelda Nintendo Game & Watch. Why?

I wanted to play Pokémon games on the go while taking advantage of the GNW's massive 6-8 hours of battery life, as well as its quick resume functionality.

I already modded the GNW yesterday using AI and about 8 hours of my time. Because of some bugs, which may have been caused by mindlessly copying and pasting every command the AI printed out, I am going to redo the entire process of flashing the firmware and emulator and document everything along the way.

### PLEASE do not use this repo as a guide for modding a GNW yourself. I already messed up my device on the first attempt by bricking the default firmware before I was able to back it up.
### Unlocking the flash erased everything.

---

## 1. Setting up the programmer

~~I am using a **Raspberry Pi Zero W** (the old version). With its single core, it's not the fastest machine for compiling C code, but given enough time, it works well enough. I set it up with the 32-bit Raspberry Pi OS Lite image and enabled SSH.~~

My RPI0 is no longer being used and has been replaced by my RPI4 (8 GB). The RPI0 crashed too many times during package installations and was generally not fun to work with because of its limited performance.

The RPI4 might be a bit overkill, but it should help later when building packages.

I installed Raspberry Pi OS Lite (64-bit), set up Wi-Fi, and enabled password-based SSH since this Pi will not be running any important services.

First, let's update the Pi:

```bash
sudo apt update && sudo apt upgrade -y
```

Next, I check the installed Python version. According to the GNWManager installation guide, we need Python >= 3.9.

```bash
python --version
```

This shows that version 3.13.5 is already preinstalled.

---

## 2. Soldering

![RPI_GNW_PINOUT](https://github.com/Nico-Hei/GNW-Modding/blob/main/Images/RPI_GNW_Pinout.png)
*Image source: https://pypi.org/project/gnwmanager/*

I own the Zelda version. As you can see, the pin layout is the same. The only difference is that the Zelda version comes with 4 MB of available storage, while the Mario version only has 1 MB.

I soldered two GPIO wires to the SWDIO and SWCLK pads without any issues. Unfortunately, I accidentally destroyed the ground pad on the debug header, so I ended up soldering the ground wire directly to the USB-C port instead.

![GNWSetup](https://github.com/Nico-Hei/GNW-Modding/blob/main/Images/GNWSetup.jpg)

### Disconnect the battery while soldering!

---

## 3. Setting up *gnwmanager*

We need GNWManager because it provides a lot of useful tools, such as flashing the GNW's memory banks and unlocking the bootloader.

SSH into the programmer.

First, install pipx:

```bash
sudo apt install pipx -y
```

(pipx is basically pip with built-in virtual environments for Python applications. It does not replace pip.)

After that:

```bash
pipx install gnwmanager
```

To add GNWManager to the system PATH:

```bash
pipx ensurepath
```

Then log out and back in.

I think OpenOCD is mainly used to unlock the GNW's bootloader, but we need to install it because GNWManager depends on it:

```bash
gnwmanager install openocd
```

Run:

```bash
gnwmanager info
```

If the following message appears, the installation was successful.

![RPI_GNW_Installation](https://github.com/Nico-Hei/GNW-Modding/blob/main/Images/RPI_GNW_Installation.png)

The error is expected because the Game & Watch is not connected to the Pi yet.

---

## 4. Connecting

Connect your GNW to the GPIO pins on your Raspberry Pi as shown in the soldering diagram.

Then either reconnect the battery or power the GNW via USB-C.

**If you get weird errors during any of the next steps, check whether the screen is on. The GNW needs to be powered on for most of this process.**

Make sure `gpiod` is installed.

When I run:

```bash
gnwmanager info
```

I still get the same error as before.

Running:

```bash
GNWMANAGER_VERBOSITY=debug gnwmanager info
```

produces the following error:

![](https://github.com/Nico-Hei/GNW-Modding/blob/main/Images/GNWError.png)

OpenOCD couldn't find the GPIO pins using what I believe is a special `sysfsgpio` configuration.

Using:

```bash
ls /usr/share/openocd/scripts/interface/
```

I can see that `sysfsgpio-raspberrypi.cfg` exists.

From my first attempt, I remembered that I actually needed to use `raspberrypi-swd.cfg`.

This suggested that I had to define the interface configuration manually:

```bash
openocd -f interface/raspberrypi-swd.cfg
```

However, OpenOCD also needed a target configuration, so I tried:

```bash
openocd -f interface/raspberrypi-swd.cfg -c "transport select swd"
```

This still wasn't enough to connect to the GNW.

Through trial and error, I discovered that I also needed:

```bash
openocd -f interface/raspberrypi-swd.cfg -c "transport select swd" -f target/stm32h7x.cfg
```

Please don't ask me what exactly `stm32h7x.cfg` is. My best guess is that it's a configuration file for the GNW's CPU.

After running this command, I was finally able to communicate directly with the GNW.

![GNWConnected](https://github.com/Nico-Hei/GNW-Modding/blob/main/Images/GNWConnected.png)

As you can see, the CPU is detected correctly:

```text
Cortex-M7 r1p1 processor detected
```

and the connection is ready to use:

```text
Examination succeeded
```

Now GNWManager has to be configured accordingly.

I found that pipx stored GNWManager's OpenOCD data in:

```text
.local/share/pipx/venvs/gnwmanager/lib/python3.13/site-packages/gnwmanager/ocdbackend/
```

Inside:

```text
openocd_backend.py
```

I found the Raspberry Pi interface settings and changed them to:

```python
cmd = base_cmd.copy()
cmd.extend(["-c", "adapter speed 1000"])
cmd.extend(["-f", "interface/raspberrypi-swd.cfg"])
cmd.extend(["-c", "transport select swd"])
cmd.extend(["-f", "target/stm32h7x.cfg"])
yield "rpi-gpio", cmd
```

*File: openocd_backend.py*

After that:

```bash
GNWMANAGER_VERBOSITY=debug gnwmanager info
```

finally produced the expected output.

---

## 5. Backups and Bootloader Unlock

I already completed this step, but normally you should now run:

```bash
gnwmanager unlock
```

and store the generated backup files in a safe place.

---

## 6. Retro-Go Installation

Now we are going to install Retro-Go as our firmware since it provides the emulators we need.

I am not promoting or encouraging game piracy. Please obtain your game backups legally.

Clone the Retro-Go repository along with its submodules:

```bash
git clone --recurse-submodules https://github.com/sylverb/game-and-watch-retro-go -b filesystem_wip
```

This branch prevents unused emulators from being compiled every time, which saves both storage space and build time.

I first had to install Git:

```bash
sudo apt install git
```

Enter the project directory:

```bash
cd game-and-watch-retro-go
```

Install pip:

```bash
sudo apt install python3-pip -y
```

Install venv:

```bash
sudo apt install python3-venv -y
```

Create a virtual environment:

```bash
python3 -m venv venv
```

Activate it:

```bash
source venv/bin/activate
```

(Use `deactivate` to leave it.)

Install the requirements:

```bash
python3 -m pip install -r requirements.txt
```

Install the ARM GCC toolchain (version 10 or newer):

```bash
sudo apt install gcc-arm-none-eabi
```

---

## 7. Adding Games and Flashing Retro-Go

Now add your ROMs to Retro-Go.

The Mario version has 1 MB of external storage, while the Zelda version has 4 MB. If you want to store multiple games, you'll probably need to research compression options.

![GNWRoms](https://github.com/Nico-Hei/GNW-Modding/blob/main/Images/GNWRoms.png)

Stay inside your Retro-Go virtual environment.

⚠️ **DO NOT RUN THIS BLINDLY**

I had to reset my flash memory with:

```bash
gnwmanager erase all
```

because I ran into storage-related issues.

Build the firmware:

```bash
make clean
```

```bash
make -j4 GNW_TARGET=zelda
```
*Zelda Theme*

or

```bash
make -j4 GNW_TARGET=mario
```
*Mario Theme*

or

```bash
make -j4 EXTFLASH_SIZE_MB=4
```
*Mario Theme on Zelda Console(Expanded storage capacity)*

Leave the virtual environment:

```bash
deactivate
```

Flash the firmware:

```bash
gnwmanager flash bank1 build/gw_retro_go_intflash.bin
```

Flash the game storage:

```bash
gnwmanager flash ext build/gw_retro_go_extflash.bin
```

---

## 8. Possible Additions

1. SD card slot and support
2. Larger flash chips (up to 64 MB)
   - Homebrew support
3. Programming the GNW via the USB-C port
4. Cover art and custom UI designs
