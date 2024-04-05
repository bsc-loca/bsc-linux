# **BSC Linux**

This repository contains the files needed to compile the Linux kernel and the OpenSBI bootloader for the BSC processor designs, in order to boot with them on an FPGA board.

The generation of the Linux filesystem and compilation is done using Buildroot. For information about Buildroot beyond the scope of this documentation, please visit the [Buildroot website](https://buildroot.org/).

This BSC Linux version is still experimental and under development. Please report any issues you find in the [issue tracker](https://gitlab.bsc.es/hwdesign/bsc-linux/-/issues/).

## **Table of Contents**

* [**Before Starting**](#before-starting)
* [**Quick Start**](#quick-start)
* [Overview](#overview)
  * [*Project structure*](#project-structure)
  * [*Boot process*](#boot-process)
* [**Customization**](#customization)
  * [*Adding custom files*](#adding-custom-files)
  * [*Fiddling with Buildroot configuration and adding packages*](#fiddling-with-buildroot-configuration-and-adding-packages)
  * [*Fiddling with Linux configuration*](#fiddling-with-linux-configuration)
* [**How to contribute**](#how-to-contribute)
  * [*Overriding default configuration*](#overriding-default-configuration)
  * [*Adding new configurations*](#adding-new-configurations)

---

## **Before Starting**

First of all, pull `buildroot` submodule:

```bash
git submodule update --init
```

**From now on, any command is assumed to be run from the `buildroot` directory.**

Then, run the following code to configure buildroot according to your required setup.

- For Sargantana core in Alveo boards (U280, U55C):

    ```bash
    make BR2_EXTERNAL=../bsc_tree/ sargantana_alveo_defconfig
    ```

More configurations will be added in the future.

## **Quick Start**

To generate the Linux filesystem and compile the Linux kernel and OpenSBI, run the following commands:

```bash
make
make linux-rebuild-with-initramfs && make opensbi-rebuild
```

The generated files will be located in the `output/images` directory. The bootable images are `fw_payload.elf` and `fw_payload.bin` (you will need this last one for FPGA).

The first time you run `make`, it will take a while to download and compile all the packages (arround 30 minutes). Subsequent runs will be way faster. Buildroot will try to paralelize compilation as much as possible, no `make -jX` is needed.

---

## Overview

The BSC Linux is based on Buildroot, a tool that allows to generate a Linux filesystem and compile the Linux kernel and OpenSBI bootloader.
From the compilation we will obtain a bootable image of Linux that can be loaded on an FPGA board.

The default configuration aims to be minimal in both size and compilation time, trying to include the minimum amount of drivers and packages, although there are some "leisure" packages for demo purposes. *(However, there could still be REALLY unnecessary stuff. If you find something that could be removed, specially drivers, do it! :D)*

### *Project structure*

The project structure is the following:

```text
.
├── buildroot
├── bsc_tree
│   ├── board
│   │   ├── sargantana_alveo
│   │   │   ├── sargantana.dts
│   │   │   ├── linux.conf
│   │   │   └── rootfs_overlay
│   │   │       └── ...
│   │   └── ...
│   ├── Config.in
│   ├── configs
│   │   ├── sargantana_alveo_defconfig
│   │   └── ...
│   ├── external.desc
│   ├── external.mk
│   └── patches
│       ├── sargantana_alveo
│       │   ├── opensbi
│       │   │   └── 0001-opensbi-sargantana-alveo-platform.patch
│       │   ├── <package name>
│       │   │   ├── 000X-package-patch_name.patch
│       │   │   └── ...
│       │   └── ...
│       └── ...
└── README.md
```

The custom configurations are located in the `bsc_tree` directory, that will act as an external tree for Buildroot. It contains the configuration files and patches needed to compile the Linux kernel and OpenSBI for the specific implementation and FPGA board.

- `bsc_tree/board/<your setup>/<name>.dts`: Device Tree file for the system.
- `bsc_tree/board/<your setup>/linux.conf`: Linux kernel configuration file.
- `bsc_tree/board/<your setup>/rootfs_overlay`: Directory containing the files that will be copied to the root of the Linux filesystem.
- `bsc_tree/configs/<your setup>_defconfig`: Buildroot configuration file.
- `bsc_tree/patches/<your setup>`: Directory containing patches that will be applied to the software on the image.
  - `bsc_tree/patches/<your setup>/opensbi/0001-opensbi-XXX-platform.patch`: Patch to add the specific platform to OpenSBI.

### *Boot process*

In rough terms, the boot process is the following:

1. The binary is loaded into the FPGA memory.
2. The FPGA boots the OpenSBI bootloader.
3. OpenSBI jumps to the Linux kernel.
4. Linux boots and loads the root filesystem.
5. Linux executes the init script, in this case BusyBox.
6. We are presented with a login prompt. We can login as root with no password.
7. Profit! (you are now in a Bash shell, by default)

## **Customization**

The changes you make in the configuration will be local to your machine. If you want to save them, refer to the [How to contribute](#how-to-contribute) section below.

### *Adding custom files*

At `bsc_tree/board/<your setup>/rootfs_overlay` you can find a mirror of the root filesystem. Any file you add to this directory will be copied (or overwritten) to the root of the Linux filesystem, mantaining the directory structure. This is useful to add custom files to the Linux filesystem.

Some files are already included, so you can use them as a reference.

Any time you add or modify files in this directory, you will need to run:

```bash
make
make linux-rebuild-with-initramfs && make opensbi-rebuild
```

This will rebuild the Linux kernel with the new files, and then rebuild OpenSBI with the new Linux kernel as payload.

(Most times you will only need to run `make linux-rebuild-with-initramfs && make opensbi-rebuild`, but sometimes Buildroot has its quirks and it's better to be safe than sorry. `make` shouldn't take more than a few seconds if nothing has changed.)

### *Fiddling with Buildroot configuration and adding packages*

There is a lot of configuration options available in Buildroot. To access them, run:

```bash
make menuconfig
```

This will open a menu where you can navigate through the different options. You can also use the search function to find the option you want to modify.

Also, Buildroot provides a curated list of commonly used packages that can be added to the image. Inside the config menu, navigate to `Target packages` to see the list of available packages. You can also use the search function to find the package you want to add.

On the matter of packages, Buildroot allows for the addition of custom packages, so those can be managed through the configuration menu. However, this is unexplored territory for us, so we can't provide any guidance on this matter. If you need to add custom software to the image, refer to [Adding custom files](#adding-custom-files) section above.

### *Fiddling with Linux configuration*

The Linux kernel configuration is managed through Buildroot. To access the configuration menu, run:

```bash
make linux-menuconfig
```

This will open a menu where you can navigate through the different options. You can also use the search function to find the option you want to modify.

---

## **How to contribute**

Please open a merge request if you have a new configuration or a fix for an existing one. We will be happy to review it and merge it into the main branch.

### *Overriding default configuration*

If you feel that you have made changes to the configuration that should override the default configuration for your board, you can save them in the corresponding `defconfig`.

If the changes are in the **Buildroot** configuration, run:

```bash
make savedefconfig
```

If the changes are in the **Linux** kernel configuration, run:

```bash
make linux-update-defconfig
```

This will save the changes in the corresponding `defconfig` file.

### *Adding new configurations*

If you want to add a new configuration, you can do so by replicating the existing structure in `bsc_tree`.
Specifically, you will need to copy (and rename accordingly) the following files and directories:

- `bsc_tree/board/sargantana_alveo` → **`bsc_tree/board/<your setup>`**
- `bsc_tree/configs/sargantana_alveo_defconfig` → **`bsc_tree/configs/<your setup>_defconfig`**
- `bsc_tree/patches/sargantana_alveo` → **`bsc_tree/patches/<your setup>`**

Then, change your current configuration to the new one:

```bash
make BR2_EXTERNAL=../bsc_tree/ <your setup>_defconfig
```

You can then modify the configuration as you wish.
