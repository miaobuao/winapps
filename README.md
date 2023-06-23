# WinApps for Linux

Run Windows apps such as Microsoft Office/Adobe in Linux (Ubuntu/Fedora) and GNOME/KDE as if they were a part of the native OS, including Nautilus integration for right clicking on files of specific mime types to open them.

<img src="demo/demo.gif" width=1000>

***Proud to have made the top spot on [r/linux](https://www.reddit.com/r/linux) on launch day.***

## How it works
WinApps was created as an easy, one command way to include apps running inside a VM (or on any RDP server) directly into GNOME as if they were native applications. WinApps works by:
- Running a Windows RDP server in a background VM container
- Checking the RDP server for installed applications such as Microsoft Office
- If those programs are installed, it creates shortcuts leveraging FreeRDP for both the CLI and the GNOME tray
- Files in your home directory are accessible via the `\\tsclient\home` mount inside the VM
- You can right click on any files in your home directory to open with an application, too

## Currently supported applications
### WinApps supports ***ANY*** installed application on your system.

It does this by:
1. Scanning your system for offically configured applications (below)
2. Scanning your system for any other EXE files with install records in the Windows Registry

Any [officially configured applications](./docs/default-apps.md) will have support for high-resolution icons and mime types for automatically detecting what files can be opened by each application. Any other detected executable files will leverage the icons pulled from the EXE.

## Installation

### Step 1: Set up a Windows Virtual Machine
The best solution for running a VM as a subsystem for WinApps would be KVM. KVM is a CPU and memory-efficient virtualization engine bundled with most major Linux distributions. To set up the VM for WinApps, follow this guide:

- [Creating a Virtual Machine in KVM](docs/KVM.md)

If you already have a Virtual Machine or server you wish to use with WinApps, you will need to merge `install/RDPApps.reg` into the VM's Windows Registry. If this VM is in KVM and you want to use auto-IP detection, you will need to name the machine `RDPWindows`. Directions for both of these can be found in the guide linked above.

### Step 2: Download the repo and prerequisites
To get things going, use:
``` bash
sudo apt-get install -y freerdp2-x11
git clone https://github.com/miaobuao/winapps.git
cd winapps
```
Arch/Linux
``` bash
sudo pacman -S freerdp
git clone https://github.com/Fmstrat/winapps.git
cd winapps
```

### Step 2: Creating your WinApps configuration file
You will need to create a `~/.config/winapps/winapps.conf` configuration file with the following information in it:
``` bash
RDP_USER="MyWindowsUser"
RDP_PASS="MyWindowsPassword"
#RDP_DOMAIN="MYDOMAIN"
#RDP_IP="192.168.123.111"
#RDP_SCALE=100
# Uncomment the line below if using a French (AZERTY) keyboard layout:
#RDP_FLAGS="/kbd:0x0000040C"
# Run `xfreerdp /kbd-list` in a terminal to see a list of keyboard layout codes.

#MULTIMON="true"
#DEBUG="true"
#VIRT_MACHINE_NAME="machine-name"
#VIRT_NEEDS_SUDO="true"
#RDP_SECRET="account"
```

#### Using Secret Tool for RDP_PASS

You can add the RDP password for lookup using secret tool. Label can be whatever you want it to be.

```bash
secret-tool store --label='winapps' winapps account
```

This will be useful if you wanna know more about keyboard layout: [FreeRDP-User-Manual#input](https://github.com/awakecoding/FreeRDP-Manuals/blob/master/User/FreeRDP-User-Manual.markdown#input).

The username and password should be a full user account and password, such as the one created when setting up Windows or a domain user. It cannot be a user/PIN combination as those are not valid for RDP access.

Options:
- When using a pre-existing non-KVM RDP server, you can use the `RDP_IP` to specify it's location
- If you are running a VM in KVM with NAT enabled, leave `RDP_IP` commented out and WinApps will auto-detect the right local IP
- For domain users, you can uncomment and change `RDP_DOMAIN`
- On high-resolution (UHD) displays, you can set `RDP_SCALE` to the scale you would like [100|140|160|180]
- To add flags to the FreeRDP call, such as `/audio-mode:1` to pass in a mic, use the `RDP_FLAGS` configuration option
- For multi-monitor setups, you can try enabling `MULTIMON`, however if you get a black screen (FreeRDP bug) you will need to revert back
- If you enable `DEBUG`, a log will be created on each application start in `~/.local/share/winapps/winapps.log`

### Step 3: Setting up your Windows VM

#### Option 1 - Running KVM
You can refer to the [KVM](https://www.linux-kvm.org) documentation for specifics, but the first thing you need to do is set up a Virtual Machine running Windows 10 Professional (or any version that supports RDP). First, clone WinApps and install KVM and FreeRDP:
``` bash
sudo apt-get install -y virt-manager
```
Arch/Manjaro
``` bash
sudo pacman -S virt-manager virt-viewer dnsmasq vde2 bridge-utils openbsd-netcat ebtables iptables
```
Now set up KVM to run as your user instead of root and allow it through AppArmor (for Ubuntu 20.04 and above):
``` bash
sudo sed -i "s/#user = "root"/user = "$(id -un)"/g" /etc/libvirt/qemu.conf
sudo sed -i "s/#group = "root"/group = "$(id -gn)"/g" /etc/libvirt/qemu.conf
sudo usermod -a -G kvm $(id -un)
sudo usermod -a -G libvirt $(id -un)
sudo systemctl restart libvirtd
sudo ln -s /etc/apparmor.d/usr.sbin.libvirtd /etc/apparmor.d/disable/

sleep 5

sudo virsh net-autostart default
sudo virsh net-start default
```
**You will likely need to reboot to ensure your current shell is added to the group.**

Next, define a VM called RDPWindows from the sample XML file with:
``` bash
virsh define kvm/RDPWindows.xml
virsh autostart RDPWindows
virsh start RDPWindows
```
Arch/Manjaro
``` bash
virsh define kvm/RDPWindowsArch.xml
virsh autostart RDPWindows
virsh start RDPWindows
```

To increase performance of the VM and decrease resource utilization, read the [Improving Performance](#improving-performance) section.

You will now want to change any settings on the VM and install Windows and whatever programs you would like, such as Microsoft Office. If the definition fails, you can always manually create a VM. You can access VMs with:
``` bash
virt-manager
```
Arch/Manjaro

Options -> File -> Add connection... -> Hypervisor: QEMU/KVM
check AutoConnect
Generated URI: qemu:///system
Connect

#### Option 2 - I already have an RDP server or VM
If you already have an RDP server or VM, using WinApps is very straight forward. Simply skip to step 4!

### Step 4: Configuring your Windows VM
After the install process, or on your current RDP server, you will want to:
- Go to the Start Menu
    - Type "About"
    - Open "About"
    - Change the PC name to "RDPWindows" if you are using KVM (This will allow WinApps to detect the local IP)
- Go to Settings
    - Under "System", then "Remote Desktop" allow remote connections for RDP
- Merge `install/RDPApps.reg` into the registry to enable RDP Applications

### Install virtio-win driver
The virtual machine uses virtio components, Windows needs the drivers for these devices. 

To install you need:

- Download the iso: virtio-win-0.1.xxx.iso (you can find it at fedorapeople.org)
- Mount the iso on the SATA CD-rom device from virt-manager
- In windows open the mounted disk and install: virtio-win-guest-tools.exe

### Step 5: Connect GNOME/KDE to your Windows VM with shortcuts and file associations
Lastly, check that FreeRDP can connect with:
```
bin/winapps check
```
You will see output from FreeRDP, as well as potentially have to accept the initial certificate. After that, a Windows Explorer window should pop up. You can close this window and press `Ctrl-C` to cancel out of FreeRDP.

If this step fails, try restarting the VM, or your problem could be related to:
- You need to accept the security cert the first time you connect (with 'check')
- Not enabling RDP in the Windows VM
- Not being able to connect to the IP of the VM
- Incorrect user credentials in `~/.config/winapps/winapps.conf`
- Not merging `install/RDPApps.reg` into the VM

Then the final step is to run the installer which will prompt you for a system or user install:
``` bash
./installer.sh
```
This will take you through the following process:

<img src="demo/installer.gif" width=1000>


## Adding pre-defined applications
Adding applications with custom icons and mime types to the installer is easy. Simply copy one of the application configurations in the `apps` folder, and:
- Edit the variables for the application
- Replace the `icon.svg` with an SVG for the application (appropriately licensed)
- Re-run the installer
- Submit a Pull Request to add it to WinApps officially

When running the installer, it will check for if any configured apps are installed, and if they are it will create the appropriate shortcuts on the host OS.

## Running applications manually
WinApps offers a manual mode for running applications that are not configured. This is completed with the `manual` flag. Executables that are in the path do not require full path definition.
``` bash
./bin/winapps manual "C:\my\directory\executableNotInPath.exe"
./bin/winapps manual executableInPath.exe
```

## Checking for new application support
The installer can be run multiple times, so simply run the below again and it will remove any current installations and update for the latest applications.
``` bash
./installer.sh
```

## Optional installer command line arguments
The following optional commands can be used to manage your application configurations without prompts:
``` bash
./installer.sh --user                # Configure applications for the current user
./installer.sh --system              # Configure applications for the entire system
./installer.sh --user --uninstall    # Remove all configured applications for the current user
./installer.sh --system --uninstall  # Remove all configured applications for the entire system
```

## Common issues
- **Black window**: This is a FreeRDP bug that sometimes comes up. Try restarting the application or rerunning the command. If that doesn't work, ensure you have `MULTIMON` disabled.

## Shout outs
- Some icons pulled from
  - Fluent UI React - Icons under [MIT License](https://github.com/Fmstrat/fluent-ui-react/blob/master/LICENSE.md)
  - Fluent UI - Icons under [MIT License](https://github.com/Fmstrat/fluentui/blob/master/LICENSE) with [restricted use](https://static2.sharepointonline.com/files/fabric/assets/microsoft_fabric_assets_license_agreement_nov_2019.pdf)
  - PKief's VSCode Material Icon Theme - Icons under [MIT License](https://github.com/Fmstrat/vscode-material-icon-theme/blob/master/LICENSE.md)
  - DiemenDesign's LibreICONS - Icons under [MIT License](https://github.com/Fmstrat/LibreICONS/blob/master/LICENSE)

