rpi-cross
=========

There are plenty of good things to say about the [Raspberry Pi](http://www.raspberrypi.org/help/what-is-a-raspberry-pi/), but there’s one thing I could do without: Waiting for compilation! The venerable Pi’s ARM11 is showing its age …

This project is meant to help: Using [VirtualBox](https://www.virtualbox.org/), [Vagrant](https://github.com/mitchellh/vagrant), [crosstool-NG](http://crosstool-ng.org/), and [Ansible](https://github.com/ansible/ansible), `rpi-cross` will automatically set up an Ubuntu VM with a cross compilation toolchain optimized for Raspberry Pi and a [distcc](https://code.google.com/p/distcc/) server, ready to cross compile both locally and over the network.

Sound good? Let’s make it happen. Contributions very welcome.

Status
------

`rpi-cross` is a work-in-progress. Right now, testers are needed to confirm the following features:

- [x] setup of a prebuilt toolchain
- [x] unattended toolchain compilation using crosstool-NG
- [x] distcc setup and configuration

All of these work for me, but I’m sure there are plenty of kinks to work out. Whatever your experience, please do provide feedback! :)

Usage
-----

### Preparation

First of all, install [VirtualBox](https://www.virtualbox.org/wiki/Downloads), [Vagrant](http://www.vagrantup.com/downloads), and [Ansible](http://docs.ansible.com/intro_installation.html). (On Mac OS, all three are available as homebrews/casks.)

Then clone the `rpi-cross` repo:

```sh
cd some/folder
git clone https://github.com/tjanson/rpi-cross.git
cd rpi-cross
```

Now, please follow along to get an overview of what’s about to happen.

### Vagrantfile and Ansible Playbook

First, take a look at the [Vagrantfile](https://github.com/tjanson/rpi-cross/blob/master/Vagrantfile) (e.g., `nano Vagrantfile`) and read the comments. Vagrant is a tool that’ll set up a headless VM (that is, a virtual machine without graphical interface) based on these settings.

Done? We’ll continue our tour with `provisioning/playbook.yml`. This is the meats: a list of state descriptions and commands that Ansible will execute automatically during “provisioning”.

These tasks, listed under [`tasks` (line 40)](https://github.com/tjanson/rpi-cross/blob/master/provisioning/playbook.yml#L40), perfom three functions (besides some basic system maintenance):

- installing crosstool-NG and building a toolchain based on a CTNG config file
- unpacking a prebuilt toolchain
- installing and configuring distcc

You’ll only want *either* build your own toolchain, *or* use a prebuilt one (and you’ll be prompted to choose during provisioning).
Variables starting with `ctng` relate to the former, `prebuilt_xt` to the latter — it’s safe to ignore whatever doesn’t apply to you.

If you don’t know what any of this means, the following should suffice:
A toolchain is a set of programs working together to compile software for a certain target architecture, in this case our Raspberry Pi. One of these tools is the `gcc` compiler, which you probably use on your Pi, but quite a few others are needed in the background.
The [crosstool-NG docs](http://crosstool-ng.org/hg/crosstool-ng/raw-file/069f43a215cc/docs/9%20-%20How%20is%20a%20toolchain%20constructed.txt) explain toolchains in detail.

Unpacking the `gcc-linaro-arm-linux-gnueabihf-raspbian.201402.tar.gz` is the most conservative option (and, incidentally, fairly convenient). We’ll use it for this guide.

Back at the top of the file, the `vars` section defines various paths and source files. (Again: The `ctng` options are for custom building only.)

Select the official-ish Linaro/Raspbian toolchain from February 2014 by uncomment the corresponding `prebuilt_xt: ...` line, and commenting out the other three:

```yml
# prebuilt_xt: "/vagrant/prebuilt-toolchains/linaro-arm-linux-gnueabihf-raspbian.201408.modified.tar.xz"
  prebuilt_xt: "/vagrant/prebuilt-toolchains/gcc-linaro-arm-linux-gnueabihf-raspbian.201402.tar.gz"
# prebuilt_xt: "/vagrant/prebuilt-toolchains/gcc-linaro-arm-linux-gnueabihf-4.9-2014.08_linux.tar.xz"
# prebuilt_xt: "/vagrant/prebuilt-toolchains/x-tools6h.archlinux.tar.xz"
```

### Download a prebuilt toolchain

Now download the toolchain we selected. You don’t need to unpack it.

```sh
cd prebuilt-toolchains
wget https://github.com/tjanson/rpi-cross/releases/download/v0.1/gcc-linaro-arm-linux-gnueabihf-raspbian.201402.tar.gz
cd ..
```

### VM provisioning

Call Vagrant to set up the VM, in the root directory of the project (which contains the Vagrantfile):

```sh
vagrant up
```

This will take a while and prompt you occasionally. Ideally, this will finish successfully. If you encounter any errors, please file an issue.

Okay, that’s it! You can now ssh into your VM:

```sh
vagrant ssh
```

In the new shell, the output of `which arm-linux-gnueabihf-gcc` should be `/home/vagrant/x-tools6h/bin/arm-linux-gnueabihf-gcc`, and `ls x-tools6h` should yield this (or something similar – but definitely not empty):

```sh
arm-linux-gnueabihf  bin  bin-tupleless  build.log.bz2  include  lib  libexec  share
```

In particular, `bin-tupleless` should be present (and contain symlinks to various tools). Try `./x-tools6h/bin-tupleless/gcc --version`. Does it output the `gcc` version information (numbers will vary), e.g., `gcc [...] 4.9.2 [...]`?

Finally, check `/etc/init.d/distcc status`. It should already be running.

### Hello, World!

If all that went well, you have a working cross compiler! Let’s try it: Create the following in `hello.c`:

```c
#include <stdio.h>

int main() {
    printf("Hello World");
    return 0;
}
```

Cross-compile it with `arm-linux-gnueabihf-gcc -c hello.c -o hello`, and check `file hello`:
```
hello: ELF 32-bit LSB  relocatable, ARM, EABI5 version 1 (SYSV), not stripped
```

This program won’t run in the VM — it’s for the Pi! Note the “ARM”. You may copy it to your Pi (`scp` is cool), and try it.

### Using distcc

TODO.

Project Goal
------------

My intention is to provide a beginner-friendly tool, useable without deep knowledge of the involved tools. 
Using a provisioning tool like Ansible has several advantages:

- fully automated setup and configuration,
- maintainability and adaptability,
- transparent, precise documentation of all steps involved.

You can simply treat the resulting setup as a black box, but if curiosity (or necessity) strikes, you can look at its blueprints — and change them.
