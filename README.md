# Gasket Driver for Google Coral Devices

**Note:** This repository is a fork of the official Google Coral [gasket-driver](https://github.com/google/gasket-driver). It includes modifications primarily intended to ensure compatibility with newer Linux kernels (6.14+). Please refer to the original repository for the canonical source and history. The original license applies to this fork and its modifications. See the "Kernel 6.14+ Compatibility Fixes" section below for details on changes.
---

## Kernel 6.14+ Compatibility Fixes

This section details the changes made in this fork compared to the original Google source code.

### Problem

The original driver code failed to compile on newer Linux distributions, specifically tested on:

* **OS:** Ubuntu 25.04 (Plucky Puffin)
* **Kernel:** 6.14.0-13-generic

The build process would fail with specific errors related to deprecated kernel APIs and macro usage.

### Changes Made

Two primary code modifications were implemented to resolve the compilation errors:

1.  **Use `noop_llseek`:**
    * **File:** `src/gasket_core.c`
    * **Change:** Replaced the file operation `.llseek = no_llseek,` with `.llseek = noop_llseek,`.
    * **Reason:** The `no_llseek` helper appears to be deprecated or removed in recent kernel versions. `noop_llseek` is the appropriate replacement for character devices that do not support seeking operations.

2.  **Correct `MODULE_IMPORT_NS` Syntax:**
    * **File:** `src/gasket_page_table.c`
    * **Change:** Modified `MODULE_IMPORT_NS(DMA_BUF);` to `MODULE_IMPORT_NS("DMA_BUF");`.
    * **Reason:** The `MODULE_IMPORT_NS` macro, used for importing symbols from other kernel modules (in this case, the DMA Buffer subsystem), now requires its namespace argument to be passed as a string literal in the kernel build system used with kernel 6.14.

### Status

With these patches applied, the driver successfully compiles using `make` in the `src` directory and can also be packaged into a DKMS `.deb` file on the specified Linux 6.14 environment. The resulting kernel modules (`gasket.ko` and `apex.ko`), whether installed manually or via DKMS, have been tested to load correctly (e.g., using `sudo modprobe gasket apex`), and the associated device nodes (e.g., `/dev/apex_0`) are created as expected.

---

## Original README.md

 Coral Gasket Driver

The Coral Gasket Driver allows usage of the [Coral EdgeTPU](https://coral.ai/) on Linux systems. The driver contains two modules:

* Gasket: Gasket (Google ASIC Software, Kernel Extensions, and Tools) is a top level driver for lightweight communication with Google ASICs.
* Apex: Apex refers to the [EdgeTPU v1](https://coral.ai/technology)

This repo contains both the source for direct integration into a kernel tree as well as the necessary files to generate a Debian DKMS package.

## Building Debian DKMS pacakge

From the top level directory, execute:

```
debuild -us -uc -tc -b
```

---

## Building and Installing (Fork Modifications for Kernel 6.14+)

Choose one of the following methods to build and install the driver from this modified source code:

### Method 1: Manual Installation

This method is simpler for quick testing but **will not automatically rebuild** the driver if your kernel is updated. You will need to repeat these steps after kernel upgrades.

1.  `cd src`
2.  `make clean`
3.  `make`
4.  `cd ..`
5.  `sudo mkdir -p /lib/modules/$(uname -r)/extra`
6.  `sudo cp src/gasket.ko src/apex.ko /lib/modules/$(uname -r)/extra/`
7.  `sudo depmod -a`
8.  `sudo modprobe gasket`
9.  `sudo modprobe apex`

### Method 2: DKMS Package Installation (Preferred on Debian/Ubuntu)

This method uses DKMS (Dynamic Kernel Module Support) to automatically rebuild and install the driver modules whenever your kernel is updated.

1.  **Install Prerequisites (if needed):** You need tools to build Debian packages and DKMS itself.
    ```bash
    sudo apt-get update
    sudo apt-get install build-essential debhelper dkms devscripts fakeroot
    ```
2.  **Navigate to Repo Root:** Go to the main directory where you cloned this repository (e.g., `cd ~/git/gasket-driver-test`).
3.  **Build the Package:** Run the `debuild` command. The flags `-us -uc` skip signing the package, `-tc` cleans before building, and `-b` creates a binary-only package.
    ```bash
    debuild -us -uc -tc -b
    ```
    *(Note: You may see warnings about `Deprecated feature: REMAKE_INITRD` or a `lintian` warning about `bad-distribution-in-changes-file unstable`; these are generally safe to ignore for this package.)*
4.  **Install the Package:** The command above creates a `.deb` file (e.g., `gasket-dkms_1.0-18_all.deb`) in the *parent* directory (`../`). Install it using `dpkg`:
    ```bash
    sudo dpkg -i ../gasket-dkms_*.deb
    ```
    The installation process will automatically build the modules for your current kernel using DKMS, install them, and run `depmod`. The modules (`gasket.ko`, `apex.ko`) should load automatically or can be loaded with `sudo modprobe gasket apex`. DKMS will handle future kernel updates.

---

## Optional: Persistent Device Permissions (udev)

By default, the `/dev/apex_N` device nodes might be created with permissions that only allow root access. To allow non-root users (belonging to a specific group) to access the Coral TPU device directly (e.g., for running TensorFlow Lite applications), you can create a `udev` rule. This method ensures the correct permissions are set automatically every time the device is detected (e.g., on boot or module load), which is more robust than manually changing permissions after each boot.

1.  **Create a dedicated group:** Create a group that will own the device node. A common name is `apex`.
    ```bash
    sudo addgroup apex
    ```

2.  **Add your user to the group:** Replace `your_username` with your actual Linux username. Add any other users who need access.
    ```bash
    sudo usermod -aG apex your_username
    ```
    **Important:** You will need to **log out and log back in** for this group membership change to take full effect for your user session.

3.  **Create the udev rule file:** Create a file in `/etc/udev/rules.d/`. The filename conventionally starts with a number (for ordering) and ends with `.rules`. Using `80-coral-tpu.rules` is a good choice.
    ```bash
    # You can use any text editor, or use nano, etc:
    sudo vi /etc/udev/rules.d/80-coral-tpu.rules
    ```
    Paste the following content into the file using `vi` (e.g., press `i` to insert text, paste the content, press `Esc` to exit insert mode). This rule identifies the apex devices (`SUBSYSTEM=="apex"`, `KERNEL=="apex_N"`) when they appear and sets their mode (permissions) and group ownership.
    * `MODE="0660"` grants read/write access to the owner (root) and members of the `apex` group.
    * `GROUP="apex"` sets the group ownership.
    * `TAG+="uaccess"` helps integrate with systemd-logind for access control based on active user sessions (recommended).

    The example below includes lines for both `apex_0` and `apex_1`. This is necessary for devices like the **Coral Dual Edge TPU** which present two separate TPU interfaces. **If you only have a single Edge TPU** device (M.2/PCIe card with one TPU), **you only need the first line** that references `KERNEL=="apex_0"`.

    ```udev
    SUBSYSTEM=="apex", KERNEL=="apex_0", MODE="0660", GROUP="apex", TAG+="uaccess"
    SUBSYSTEM=="apex", KERNEL=="apex_1", MODE="0660", GROUP="apex", TAG+="uaccess"
    ```
    Save and close the file (e.g., press `Esc` then type `:wq` and press `Enter` in vi).

4.  **Apply the new rule:** Tell the `udev` system to reload its rule definitions and then re-trigger events to apply rules to any existing devices.
    ```bash
    sudo udevadm control --reload-rules
    sudo udevadm trigger
    ```

Now, when the `apex` module loads (or after the trigger command if already loaded), the relevant `/dev/apex_N` device node(s) should be created with read/write permissions for the `apex` group. Users in that group (after logging back in) should be able to access the device directly without needing `sudo`. You can verify by checking `ls -l /dev/apex*`.

---

## License

This software is licensed under the terms described in the `LICENSE` file. The original license from Google applies to all code in this repository, including modifications made in this fork.

*(Generated on: Wednesday, April 9, 2025 at 12:05:35 AM PDT)*