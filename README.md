# Enabling SocketCAN on WSL2

This documentation is based on [https://gist.github.com/yonatanh20/664f07d62eb028db18aa98b00afae5a6](https://gist.github.com/yonatanh20/664f07d62eb028db18aa98b00afae5a6).

Unfortunately, when using Microsoft's WSL Kernel, SocketCAN is not enabled by default. To enable SocketCAN's can-utils on WSL we need to enable the CAN interface module in the WSL, to do so requires a re-building of the WSL kernel.  

**Requirements:**
* WSL2
* back up your wsl image (optional) follow [here (wsl2-backup-and-restore-images)](https://www.virtualizationhowto.com/2021/01/wsl2-backup-and-restore-images-using-import-and-export/)
Steps:

## Step 1: Update WSL and install can-utils

**From cmd / powershell:**
```ps
wsl --shutdown
wsl --update
```

**From wsl:**
```bash
sudo apt update
sudo apt install can-utils
```

If you try now to use candump for example you'd get this error message:

`socket: Address family not supported by protocol`

This just means that WSL2 doesn't come with the CAN interface support enabled, so we need to build the wsl kernel and enable it.

## Step 2: Get the latest WSL2 kernel and configure it for can and vcan support. 

```bash
sudo apt install build-essential flex bison libssl-dev libelf-dev libncurses-dev bc dwarves
cd ~
git clone https://github.com/microsoft/WSL2-Linux-Kernel
cd WSL2-Linux-Kernel
```

You'll then want to figure out your current WSL Kernel version by running the following command:

```bash
uname -r
``` 
This will output the current version. You'll then run `git branch -a` and match your version with the appropriate branch. In my case, my kernel version returned `5.15.167.4-microsoft-standard-WSL2+`. So I ran the following git command:

```bash
git checkout linux-msft-wsl-5.15.y
```

Now lets configure our custom kernel.

```bash
cat /proc/config.gz | gunzip > .config
make prepare modules_prepare
make menuconfig 
```
The config cli app will pop up and we need to enable the following settings.

1. Enter Networking support
2. Change **CAN bus subsystem support** to `M` and enter its submenu.
3. Change **Raw CAN Protocol** to `M`
4. Enter **CAN Device Drivers**
5. Change **Virtual Local CAN Interface** to `M`
6. (Optional) Set any devices on this list to `M` if needed.
   - **PEAK-System PCAN-PCIe FD Cards**
7. Change **Serial / USB serial CAN adapters** to `M` 
8. Make sure CAN bit-timing calculation is set to *
9. (Optional) Change CAN devices debugging messages to *
10. (Optional) Enter **CAN USB Interfaces** and enable any USB devices that you plan on using.
    - Currently our team is using the **PEAK PCAN-USB/USB Pro interfaces for CAN 2.0b/CAN-FD**.

11. Save and exit.

Here is a collection of screenshots for each of those submenus:

**Main Menu**

![Linux Kernel Configuration](https://raw.githubusercontent.com/Innovation-for-Automation/wsl-socket-can-instructions/refs/heads/main/images/Kernel-Config-1.png)

**Network Support**

![Linux Kernel Configuration - Network Support](https://raw.githubusercontent.com/Innovation-for-Automation/wsl-socket-can-instructions/refs/heads/main/images/Kernel-Config-2.png)

**CAN bus subsystem support**

![Linux Kernel Configuration - CAN bus subsystem support](https://raw.githubusercontent.com/Innovation-for-Automation/wsl-socket-can-instructions/refs/heads/main/images/Kernel-Config-3.png)

**Can Device Drivers**

![Linux Kernel Configuration - CAN Device Drivers](https://raw.githubusercontent.com/Innovation-for-Automation/wsl-socket-can-instructions/refs/heads/main/images/Kernel-Config-4.png)

**CAN USB interfaces**

![Linux Kernel Configuration - CAN USB interfaces](https://raw.githubusercontent.com/Innovation-for-Automation/wsl-socket-can-instructions/refs/heads/main/images/Kernel-Config-5.png)


## Step 3: Build and export our custom kernel

```bash
make modules
sudo make modules_install
make -j $(nproc)
sudo make install
cp vmlinux /mnt/c/Users/[USERNAME]/
```

The file vmlinux was created and now we will load it. 
We need to create config file for wsl to load the custom kernel.
Create the file `.wslconfig` in your Windows user folder C:/Users/[USERNAME]/ with the following content:

```conf
[wsl2]
kernel=C:\\Users\\USERNAME\\vmlinux
```
Now you can reset WSL with the new kenrel.

### Using SocketCAN modules on host WSL

If we are just running these modules on the host WSD system, we'll have to run the following commands every time we open the WSD terminal:

```bash
sudo modprobe can
sudo modprobe can-raw
sudo modprobe pcan
sudo modprobe vcan
sudo modprobe slcan
```

### Using SocketCAN in Dev Containers

However, if we're using these modules for specific projects in the IFA ROS/ROS2 dev containers, we can paste the following lines into the `.devcontainer/initialize.sh` file in the project.

```bash
sudo apt install kmod
sudo modprobe can
sudo modprobe can-raw
sudo modprobe pcan
sudo modprobe vcan
sudo modprobe slcan
```

And now you are able to create virtual can devices to run. For example:

```bash
sudo ip link add dev vcan0 type vcan
sudo ip link set up vcan0
sudo ip link add dev vcan1 type vcan
sudo ip link set up vcan1
```

### Sources from original documentation:

* [charliex2's comment on the Reddit thread with most of the information given here](https://www.reddit.com/r/CarHacking/comments/ot3gjf/socketcancanutils_on_windows/)
* [WSL github issue: Add SocketCAN support](https://github.com/microsoft/WSL/issues/5533)
* [weifengdq's tutorial that encompases most of this tutorial](https://chowdera.com/2022/01/202201082236197554.html)
