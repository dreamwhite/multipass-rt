# Multipass RT

## Introduction

*Q*: Why I'm creating this repository?

*A*: I'm enrolled in an IT university and for the exam "Design and development of real-time systems" we have to create an instance of a VM with a patched Linux kernel that supports RT stuff. For more infos on what is a Real Time Operating System, check out [here](https://en.wikipedia.org/wiki/Real-time_operating_system).

*Q*: What is Multipass?

*A*: Multipass is a CLI to launch and manage VMs on Windows, Mac and Linux that simulates a cloud environment with support for cloud-init.

## Requirements

Please note that I'm on `macOS Monterey 12.3 (21E230)` on a `Intel Core i5-8250U` and 16GB of RAM but you can follow the procedure even on Windows or Linux (for this last one I don't get the point of poisoning an existing installation, hence the need of creating another one).

- [multipass](https://multipass.run/) - Download and install the version for your OS, in my case macOS
- [Microsoft Remote Desktop](https://apps.apple.com/us/app/microsoft-remote-desktop/id1295203466?mt=12) or any RDP compatible application (Remmina et similia)

## Step 1: create an instance of Ubuntu 21.10 (impish)

Fire up your favourite terminal and write the below command to set `QEMU` as default driver:

`sudo multipass set local.driver=qemu`

After that create an Ubuntu 21.10 instance with 4GB of RAM, 20GB of disk space and 4 CPU cores using this command:

`multipass launch --name MyBestVM --mem 4G --disk 20G --cpus 4 impish`

After the command finishes you can use `multipass list` to check if the VM is running or not.

## OPTIONAL Step 1.a: configure RDP.

As compared to other VM managers (VirtualBox, UTM), Multipass by default doesn't have any GUI that let's you see what is goin on in your VM, hence the need of configuring a desktop environment and setting RDP.

To attach to the shell use `multipass shell MyBestVM` (change `MyBestVM` with your VM name), then run the following commands (one per line except the `#` comments):

```bash
sudo passwd #to change the root password
sudo apt update #to update the local repository with the latest available mirror infos
sudo apt install ubuntu-minimal-desktop xrdp #to install minimal Ubuntu desktop stuff and xrdp
sudo systemctl enable xrdp #to enable at the start of the VM the XRDP server
```

It may take a while to download all the trash stuff that Ubuntu carries out with but you'll have all the necessary stuff to work out with a RDP connection.

To attach to the RDP session, first of all get the Local IP address of the VM (you can use either `multipass list` or `ip addr` in the VM shell, it's the same). Then open your RDP client (in my case Microsoft Remote Desktop) and add a new connection with the following parameters:

- PC Name: `IP`
- User account:
    - User: `root` (at the moment this is the only way to get it working. I don't have much spare time to investigate on it and use a low-privileged account)
    - Password: the one you set before using `sudo passwd`

Then just save and connect to the RDP and you'll reach the desktop :D 

## Step 2: patching the kernel to support RT stuff

### Requirements:
- Compiling tools
    - `sudo apt install -y qtbase5-dev qtchooser qt5-qmake qtbase5-dev-tools patch build-essential gcc g++ make libncurses5-dev bc bison flex libssl-dev libelf-dev dwarves zstd`
- [Linux Kernel](https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.15.26.tar.gz) and [RT patch](https://mirrors.edge.kernel.org/pub/linux/kernel/projects/rt/5.15/older/patch-5.15.26-rt34.patch.gz)

### How to proceed:

I'm not a fan of GUI stuff so I'll just give a bunch of commands that will help you editing all the stuff (execute one command per line):

```bash
cd /usr/src
sudo wget https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.15.26.tar.gz
sudo wget https://mirrors.edge.kernel.org/pub/linux/kernel/projects/rt/5.15/older/patch-5.15.26-rt34.patch.gz
sudo tar -zxvf linux-5.15.26.tar.gz
sudo gzip -d patch-5.15.26-rt34.tar.gz
cd linux-5.15.26
sudo patch -p1 < ../patch-5.15.26-rt34.patch
sudo make menuconfig
```

This last command will open up a nice TUI interface that will help you configure all the stuff (I won't write the patches there, but you know where to retrieve them :D). In case you wanna configure using a GUI, run `sudo make xconfig` instead of `sudo make menuconfig`).

After doing so run the following commands to prepare the compilation process:

##### please note that `sudo make -j5` means "yo use 5-1 cores (4)". In case you have a different configuration just replace the number with n+1 CPU cores (e.g. you have 64 cores, you'll write 65)

```bash
sudo scripts/config --disable SYSTEM_TRUSTED_KEYS
sudo scripts/config --disable SYSTEM_REVOCATION_KEYS
sudo make localmodconfig
sudo make -j5
```

If everything went well, you can then proceed my installing the kernel modules, creating the bzImage and installing the kernel itself via:

```bash
sudo make modules_install
sudo make -j5 bzImage
sudo make install
```

Finally just reboot your VM (using `sudo reboot` if connected via shell or `multipass restart MyBestVM`) and after attaching to it (using RDP or shell) run `uname -a`: you'll see `PREEMPT-RT`

# Bugs

Apparently on macOS, QEMU has a weird bug that will double the amount of used RAM. In case you have limited RAM, set up 3GB by editing `--mem xG` (where `x` stands for the amount of RAM GBs).

From what I'm experiencing, 6GB of memory will be used even if you setup 3GB. And for M1 users with just 8GB, it's really painful... Sorry guys but this doesn't depend on me.

# Credits

- my IT teacher (idk if he has a GitHub account or not, in such case feel free to open a PR)
- [@b0rderljne](https://github.com/b0rderljne) for helping me through existential crises while setting up RDP
