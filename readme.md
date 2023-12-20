# Jetson Orin Nano NVMe flash using WSL2 and SDKManager
(Up to date as of 12/20/2023)
- This process sucked, horribly. If possible, use an existing (non VM) installation of Ubuntu 20.04. It should be easier.
- To install using WSL, you will have to do many tedious things, of which include:
  - Building the WSL2 kernel from source (eww).
  - Installing Nvidia's SDKManager.
  - Waiting several eternities for things to happen.
- For this installation, I will be selecting Jetpack 5.1.2, because Jetpack 6.0 was uncooperative and only led to a black screen on boot.
- The newest version of Ubuntu that supports Jetpack 5.1.2 is Ubuntu-20.04. Make sure you are using that version to run commands.

## Step 1
- Lets get the most tedious step out of the way first, building the WSL kernel from source.
- Ensure you have all required dependencies installed:
```
sudo apt-get update
sudo apt install build-essential flex bison dwarves libssl-dev libelf-dev bc
```
- Next, clone the WSL2 kernel found on GitHub ([source](https://github.com/microsoft/WSL2-Linux-Kernel)):
```
git clone https://github.com/microsoft/WSL2-Linux-Kernel
```
- Make sure that you clone this onto a filesystem that supports files of the same name but with different capitalizations coexisting. Basically, just clone this on your home directory in Ubuntu.
- Next, download copy this custom config and extract it ([source](https://forums.developer.nvidia.com/t/flash-jetson-orin-nano-wsl2/263654/9)). It can be downloaded via this [link](https://forums.developer.nvidia.com/uploads/short-url/NjlHVIo6tP4slqAJQND462YIk7.gz).
- Copy the `config` file found in the extracted folder into the root of the WSL kernel repository that was just cloned.
  `cp ./config ~/WSL2-Linux-Kernel`
- Start building the kernel, this will take a while.
  `make KCONFIG_CONFIG=config`
  or, for multithreaded building (with 8 threads as an example):
  `make KCONFIG_CONFIG=config -j 8`
- You may have to run the single threaded build command once the multithreaded command finishes to produce the resulting `vmlinux` binary.
- Now, copy `vmlinux` to somewhere outside of your WSL drive. I've copied mine to `C:\temp\vmlinux\wsl-kernel`:
```
mkdir -p /mnt/c/temp/wsl-kernel/
cp vmlinux /mnt/c/temp/wsl-kernel/vmlinux
```
- To use this kernel with WSL, we have to edit (or create and edit) the file `C:\Users\[user]\.wslconfig` and add:
```
[wsl2]
kernel=C:\\temp\\wsl-kernel\\vmlinux
```

## Step 2
- Install NVMe drive and SD card into the Jetson (both required)
- Bridge the two adjacent pins: FC REC and GND on the Jetson. This can be done with tweezers if you don't have a jumper.
- Plug in the Jetson Orin Nano using the usb C port. Plug the other end into your host PC.
- Download and install usbipd.exe from [here](https://github.com/dorssel/usbipd-win/releases/latest)

## Step 3
- Install the prerequisites for SDKManager:
```
sudo apt update
sudo apt install iputils-ping iproute2 netcat iptables dnsutils network-manager usbutils net-tools python3-yaml dosfstools libgetopt-complete-perl openssh-client binutils xxd cpio udev dmidecode -y
sudo apt install linux-tools-virtual hwdata -y
sudo apt install libgbm1 libgtk-3-0 libatk-bridge2.0-0 libgconf-2-4 -y
```
- Download the SDKManager `.deb` file from Nvidia's website, [here](https://developer.download.nvidia.com/sdkmanager/redirects/sdkmanager-deb.html)
- Install it
```
sudo apt install ./sdkmanager_[version]-[build#]_amd64.deb
```
- Install `qemu-user-static` and ensure the required binary formats are imported:
```
sudo apt-get install qemu-user-static
sudo update-binfmts --import qemu-aarch64
sudo update-binfmts --import qemu-arm
sudo update-binfmts --import qemu-armeb
```
- Enable systemd by appending into your `wsl.conf`:
```
[boot]
systemd=true
```
- Shutdown, then reopen wsl to ensure everything is working properly:
```
wsl --shutdown
```

## Step 4
- Power on the Jetson ***while*** the FC REC and GND pins are bridged. This will boot the Jetson into forced recovery mode. This allows us to connect via USB.
- Using powershell, run `usbipd.exe list` and confirm that a device with a `VID` starting with `0955` is present.
  - Take note of the `BUSID`.
- Share the Jetsons USB with WSL using the command (replace BUSID with your BUSID) (run as administrator):
```
usbipd.exe bind -b <BUSID> --force
```
- To make sure the USB stays connected during the flashing process, run this command to auto-reconnect it when it disconnects (once again, replace BUSID):
```
usbipd.exe attach --wsl -b <BUSID> --auto-attach
```
- In WSL, open sdkmanager via the commandline:
```
sdkmanager
```
- Login to your Nvidia account.
- The Jetson should be automatically detected as long as usbipd is running. Select the "Developer Kit" version when presented with the option.
- Select "JetPack 5.1.2" as your SDK Version.
- Unselect the "Host Machine" option from System Configuration.
- Proceed to next step.
- Accept the Nvidia License Agreement at the bottom and Continue to the next step.
- Wait for all components to download.
- Once all components have downloaded, flashing will begin.
- A popup will show up to connect to the jetson for flashing. Select `NVMe` as the storage device. Select `Pre-Configured` as the configuration, along with a username and password.
- Make sure to watch the terminal tab closely for errors. If you notice the error `Unknown device "/sys/class/net/bonding_masters": No such device`, do the following:
  - Quickly close the powershell instance running `usbipd.exe`. 
  - Reopen it again, still in administrator mode.
  - Try to run 
```
usbipd.exe bind -b <BUSID> --force
```
    until it succeeds, and then run
```
usbipd.exe attach --wsl -b <BUSID> --auto-attach
```
  - This must be done before the flashing times out.
- Once flashing has finished, the jetson will reboot. Make sure you have a monitor + peripherals to set it up.
- Another popup will show up on the SDKManager to install additional components. On the Jetson, find the ip of it using `ifconfig`. Add this information into the popup along with the username and password.
  - This step is really optional. If you don't need the extra components, you're done.
- The SDKManager will now connect with SSH and install the rest of the components.
- You're done! You can now remove the jumper from FC REC and GND and restart your Jetson.
