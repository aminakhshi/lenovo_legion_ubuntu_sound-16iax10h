# Guide: Linux Audio on the Lenovo Legion Pro 7i Gen 10 (16IAX10H) — Ubuntu

This guide explains how to get audio working correctly on the **Lenovo Legion Pro 7i Gen 10 (16IAX10H)** on Ubuntu by patching and building a custom Linux kernel.

Because this solution is still very new, it may take some time before all components are fully integrated upstream into the Linux kernel. Until then, the steps below have been **rigorously tested and confirmed to work**. This guide will be updated as future kernel versions are released until the fix is fully upstreamed.

---

## Help Needed: Upstreaming

If you have time and kernel experience, please help upstream these changes and keep us posted on your progress.  
This [Kernel Bugzilla comment](https://bugzilla.kernel.org/show_bug.cgi?id=218329#c24) contains useful pointers.

---

## Confirmed to Work on Multiple Devices

This fix has been confirmed to work on more laptops than just the 16IAX10H:

- Lenovo Legion Pro 7i Gen 10 (**16IAX10H**)
- Lenovo Legion Pro 7 Gen 10 (**[16AFR10H](https://github.com/nadimkobeissi/16iax10h-linux-sound-saga/issues/30)**)
- Lenovo Legion 5i Gen 9 (**[16IRX9](https://github.com/nadimkobeissi/16iax10h-linux-sound-saga/issues/20)**)

If your laptop uses a similar sound architecture and exhibits similar issues, please try this fix and report your results.

---

## Step 1: Install Build Dependencies (Ubuntu)

```bash
sudo apt update
sudo apt install -y \
  build-essential \
  libncurses-dev \
  flex \
  bison \
  libssl-dev \
  libelf-dev \
  dwarves \
  bc \
  rsync \
  dkms \
  git
````

---

## Step 2: Clone the Repository

```bash
git clone https://github.com/nadimkobeissi/16iax10h-linux-sound-saga.git ~/legion
```

---

## Step 3: Install the AW88399 Firmware

Copy the firmware blob provided in this repository:

```bash
sudo cp -f ~/legion/fix/firmware/aw88399_acf.bin /lib/firmware/aw88399_acf.bin
```

If you prefer to extract the firmware yourself, follow
[these instructions](https://bugzilla.kernel.org/show_bug.cgi?id=218329#c18).

---

## Step 4: Download the Linux Kernel Sources

This fix has been tested with:

* **Linux 6.18**
* Also verified on **6.18.1 – 6.18.5**

```bash
wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.18.tar.xz
tar -xf linux-6.18.tar.xz
cd linux-6.18
```

---

## Step 5: Patch the Kernel Sources

Copy the patch matching your kernel version into the kernel source root and apply it:

```bash
cp -f ~/legion/fix/patches/16iax10h-audio-linux-6.18.patch .
patch -p1 < 16iax10h-audio-linux-6.18.patch
```

The patch should apply cleanly to **10 files** with no errors.

---

## Step 6: Kernel Configuration

### 6.1 Start From Ubuntu’s Running Kernel Configuration

```bash
zcat /proc/config.gz > .config
```

### 6.2 Ensure NVMe Is Built Into the Kernel (Critical for Dual Boot)

If your root filesystem is on NVMe, it **must not be a module**, or early boot will fail.

Verify in `.config`:

```text
CONFIG_BLK_DEV_NVME=y
CONFIG_NVME_CORE=y
```

You can confirm interactively:

```bash
make menuconfig
```

Navigate to:

```
Device Drivers  --->
  NVM Express block device support  --->
    <*> NVM Express block device
```

---

## Step 7: Enable Required Audio and SOF Options

Ensure the following kernel options are enabled:

```text
CONFIG_SND_HDA_SCODEC_AW88399=m
CONFIG_SND_HDA_SCODEC_AW88399_I2C=m
CONFIG_SND_SOC_AW88399=m
CONFIG_SND_SOC_SOF_INTEL_TOPLEVEL=y
CONFIG_SND_SOC_SOF_INTEL_COMMON=m
CONFIG_SND_SOC_SOF_INTEL_MTL=m
CONFIG_SND_SOC_SOF_INTEL_LNL=m
```

### Optional: Append to Existing Config Automatically

```bash
cat >> .config <<EOF
CONFIG_SND_HDA_SCODEC_AW88399=m
CONFIG_SND_HDA_SCODEC_AW88399_I2C=m
CONFIG_SND_SOC_AW88399=m
CONFIG_SND_SOC_SOF_INTEL_TOPLEVEL=y
CONFIG_SND_SOC_SOF_INTEL_COMMON=m
CONFIG_SND_SOC_SOF_INTEL_MTL=m
CONFIG_SND_SOC_SOF_INTEL_LNL=m
EOF
```

Then finalize configuration:

```bash
make olddefconfig
```

---

## Step 8: Compile and Install the Kernel

```bash
make -j$(nproc)
make -j$(nproc) modules
sudo make modules_install
sudo make install
sudo update-grub
```

---

## Step 9: Install NVIDIA DKMS Drivers (Ubuntu Only)

If your system uses an NVIDIA GPU, you **must install the NVIDIA DKMS driver** so that kernel modules are rebuilt for your newly compiled custom kernel.

### 9.1 Identify the Recommended Driver

```bash
ubuntu-drivers devices
```

### 9.2 Install the Driver and DKMS Package

Example (replace `550` with the recommended version):

```bash
sudo apt install nvidia-driver-550 nvidia-dkms-550
```

### 9.3 Verify DKMS Status

```bash
dkms status
```

You should see something like:

```text
nvidia/550.xx.xx, <kernel-version>, x86_64: installed
```

### 9.4 (Optional) Force Rebuild if Needed

```bash
sudo dkms autoinstall
```

---

## Step 10: Generate the Initramfs (Ubuntu)

```bash
sudo update-initramfs -c -k $(cat include/config/kernel.release)
```

Ensure your bootloader includes the following kernel parameter:

```text
snd_intel_dspcfg.dsp_driver=3
```

---

## Step 11: Reboot Into the Patched Kernel

```bash
uname -a
```

Confirm you are running the newly installed kernel.

---

## Step 12: Install the Patched ALSA UCM2 Configuration

```bash
sudo cp -f ~/legion/fix/ucm2/HiFi-analog.conf /usr/share/alsa/ucm2/HDA/HiFi-analog.conf
sudo cp -f ~/legion/fix/ucm2/HiFi-mic.conf /usr/share/alsa/ucm2/HDA/HiFi-mic.conf
```

Identify your sound card:

```bash
alsaucm listcards
```

Example output:

```text
0: hw:0
  LENOVO-83F5-LegionPro716IAX10H-LNVNB161216
```

Apply the configuration (adjust card index if needed):

```bash
alsaucm -c hw:0 reset
alsaucm -c hw:0 reload
systemctl --user restart pipewire pipewire-pulse wireplumber
amixer sset -c 0 Master 100%
amixer sset -c 0 Headphone 100%
amixer sset -c 0 Speaker 100%
```

> **Important:**
> These `amixer` commands calibrate the speakers.
> They **do not** set your user volume to maximum.


## Disclaimer

THE PROGRAM IS PROVIDED “AS IS”, WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY OR FITNESS FOR A PARTICULAR PURPOSE. USE AT YOUR OWN RISK.

---

## Credits

* **Lyapsus** — ~95% of the kernel engineering work (primary author)
* **Nadim Kobeissi** — investigation, debugging, testing, documentation
* **Gergo K.** — firmware extraction methodology
* **Richard Garber** — internal microphone fix
* Everyone who pledged support — the reward goes to Lyapsus

```
```
