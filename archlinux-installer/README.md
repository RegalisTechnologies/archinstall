# archinstall

archinstall is a simple tool which allow you to automatically deploy Arch GNU/Linux systems. It is designed for automation, archinstall supports different schemes and hooks.

## Basic usage

To install basic  Arch GNU/Linux in `/tmp/arch` directory just type:
```shell
$ mkdir /tmp/arch 
$ archinstall -t /tmp/arch -s base
```

Script will download and install required packages, generate initramfs. Then, you can use `cp`, `tar`, `rsync` etc. to copy your new system to device of your choice. After this operation, you need to install bootloader and configure /etc/fstab.

## Hooks

Hook is a special group of functions, each hook is responsible for doing only one thing. For example hook `detect_architecture` will set target architecture based on `/proc/cpuinfo`.

Function naming:

 * `hook_NAME_preinstall` - will be executed before install hook
 * `hook_NAME_install` - will be executed as a main install routine
 * `hook_NAME_postinstall` - will be executed after install hook
 * `hook_NAME_desc` - will be executed when help requested

Each hook must implement at least one function. For example, internal hook `base` is responsible only for adding base packages into installation queue and it looks like this:

```shell
hook_base_preinstall() {
		info "adding base packages"
		add_packages "base base-devel grub-bios vim openssh net-tools tree";
}
```

Since we don not need to do enything else after the installation, we can ommit deifinition of `hook_base_posstinstall()`.

## Schemes

Scheme is a ordered list od hooks. It defines which hooks should be called. Example scheme:

```shell
SCHEME_base="detect_architecture base pacman prepare_chroot build_mkcinitcpio"
```

## Tuning

You can easly integrate your own hooks into `archinstall` script. For example, if you want to package your system after successfull installation you can create hook named `package` and define it like this:

```shell
hook_package_postinstall() {
	tar -C ${TARGET} -c . -vf /tmp/arch.tar
}
```

*Note: we use `postinstall` here.*

Then, the only one thing to do is to define new scheme:

```shell
SCHEME_base_package="detect_architecture base pacman prepare_chroot build_mkcinitcpio prepare_chroot_cleanup package"
```

and register it:

```shell
register_scheme base_package
```

**Note:** To generate initramfs, we need to mount `/proc`, `/sys` and `/dev`, that is why `prepare_chroot` hook is placed **before** `build_mkinitcpio`. We do not want to include /proc, /sys and /dev into our tarball so it is required to place `prepare_chroot_cleanup` before `package` hook. 

Then, you can install Arch GNU/Linux and create package with one shot:

```shell
$ archinstall -t /tmp/arch -s base_package
```

## Built-in variables, hooks and schemes

### Special variables:

 * `TARGET` - target directory
 * `ARCHITECTURE` - new system arcitecture
 * `HOOKS` - hooks scheduled for execution
 * `PACKAGES` - packages scheduled for for installation
 * `REGISTERED_SCHEMES` - list of all registered schemes
 * `INSTALL_HOOK` - current install hook

### Functions:

 * `add_packages` - schedule packages for installation
	* *$1...$n* - packages
 * `set_install_hook` - set new install hook
	* *$1* - new install hook
 * `chroot_exec` - execute program inside target's chroot environment
	* *$1* - program to execute
	* *$2...$n* - program args

## Tips and tricks

You can easly integrate archinstall with your own scripts. For example, you can use the following solution to dynamically load hooks/schemes into your script.

Directory stryucture:

	.
	├── hooks.d/
	│   ├── build_initramfs
	│   └── build_kernel
	├── schemes.d/
	│   ├── arch_pxe
	│   └── arch_usb_squashfs
	├── archinstall
	└── my_archinstall_wrapper

	2 directories, 6 files

We will use `my_archinstall_wrapper` as a wrapper around original `archinstall`:

```shell
#!/bin/bash

# Load archinstall
. archinstall

# Load hooks
for X in hooks.d/*; do 
	[[ -f $X ]] && . $X
done

# Load and register schemes:
for X in schemes.d/*; do 
	[[ -f $X ]] && . $X
	register_scheme $X
done

# Call original main routine and pass all arguments

main $@

```

**Note:** schemes must have exactly the same name as file (with `SCHEME_` prefix - see *Schemes* section). For example `schemes.d/arch_pxe` must define `arch_pxe` scheme like this:

```shell
SCHEME_arch_pxe="(...)"
```

## License

![GPLv3](http://www.gnu.org/graphics/gplv3-127x51.png)

Copyright (C) Patryk Jaworski \<regalis@regalis.com.pl\>

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License
along with this program.  If not, see http://www.gnu.org/licenses.

