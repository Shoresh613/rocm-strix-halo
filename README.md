# ROCm support for Strix Halo

Required components:

* Ubuntu 24.3 LTS
* Kernel 6.11
* ROCm v.6.4.2

Assuming you have the correct version of Ubuntu installed.

## Install the kernel

Install the kernel using mainline.
Add repo

```bash 
sudo add-apt-repository ppa:cappelikan/ppa -y
sudo apt update
sudo apt install mainline -y
```

Install kernel
```bash 
sudo mainline install 6.11.11
```


Make sure you get a boot menu to choose kernel.

```bash 
sudo nano /etc/default/grub
```

Change

Then update grub.
```bash 
sudo update-grub
```



Reboot into the new (old) kernel.
You should get a menu where you have 5 seconds to enter the Advanced Ubuntu boot options. Do that to boot into 6.11.

Make sure you are on the new kernel:

```bash 
uname -r
```
This should show 6.11*.

### Making 6.11 default

You can if you prefer make 6.11 the default kernel.

First tell grub to use a saved default:

```bash
sudo sed -i 's/^GRUB_DEFAULT=.*/GRUB_DEFAULT=saved/' /etc/default/grub
sudo sed -i 's/^#GRUB_SAVEDEFAULT=.*/GRUB_SAVEDEFAULT=true/' /etc/default/grub
```

Next, check which one you want by defalt

```bash
grep menuentry /boot/grub/grub.cfg | grep -n 'Ubuntu'
```

Note the number of the preferred entry (in this example it's 11).

Set the preferred entry as default

```bash
sudo nano /etc/default/grub
```
Update the line with GRUB_DEFAULT to `GRUB_DEFAULT="Advanced options for Ubuntu>Ubuntu, with Linux 6.11.11-061111-generic"`, replacing "Ubuntu, with Linux 6.11.11-061111-generic" with your exact version according the above.

```bash
sudo update-grub
```

After reboot, confirm with

```bash
uname -r
```

that the correct kernel is in use.

## Install ROCm


PyTorch ROCm 6.4 (and most precompiled HIP binaries) do not yet include gfx1151 (RDNA3) as a target. Therefore we need to tell ROCm to pretend the GPU is a gfx1103 or a gfx1100, which are fully supported.

Download and install the AMD GPU installer:

```bash
wget https://repo.radeon.com/amdgpu-install/6.4.2/ubuntu/noble/amdgpu-install_6.4.60402-1_all.deb
sudo dpkg -i ./amdgpu-install_6.4.60402-1_all.deb
sudo apt update
```

Run the installer:

```bash 
amdgpu-install --usecase=rocm,opencl,hip --no-dkms --accept-eula
```

Add yourself to the video and render user groups:

```bash 
sudo usermod -aG video,render $USER
```

Reboot, making sure you choose the correct kernel (6.11).

Run this to see if it finds the AMD GPU:

```bash
lsmod | grep amdgpu
```

If not, add it manually:

```bash 
sudo modprobe amdgpu
```

Now it should find it. 

You can make this permanent by telling it to always load this module:

```bash
echo amdgpu | sudo tee /etc/modules-load.d/amdgpu.conf
```

It could be that it is still blacklisted. Find out:

```bash
grep -R "blacklist amdgpu" /etc/modprobe.d
```

If something is found, you need to comment it out or remove it.

Edit the file listed in the above command, e.g.:

```bash
sudo nano /etc/modprobe.d/blacklist-amdgpu.conf
```
Comment the line out using # or delete the line.

If you did the above step, you also need to rebuild initramfs specifying your kernel:

```bash
sudo update-initramfs -u -k 6.11.11-061111-generic
```

Reboot to make sure it sticks. 

Also make sure rocminfo finds it:

```bash
/opt/rocm/bin/rocminfo | grep -A3 Name
```

Congratulations, you now have ROCm support on your Strix Halo!

You can now use `rcm-smi` to check the status of your GPU (workload, temperature). Use watch for permanent reading. In a terminal you can have open (if you like):

```bash 
watch rocm-smi
```

### Making it compatible

PyTorch ROCm 6.4 (and most precompiled HIP binaries) do not yet include gfx1151 (RDNA3 architecture) as a target.
They usually support up to gfx1100, gfx1101, and gfx1102

The fix - add permanently to .bash_rc:

```bash 
echo 'export HSA_OVERRIDE_GFX_VERSION=11.0.0' >> ~/.bashrc
```
then read the changes.

```bash 
source ~/.bash_rc
```

If 11.0.0 doesn't do the trick, try 11.0.3, which sometimes works better for Phoenix chips.

Now most things should run on your hardware.

## Try it out using Docling

To see if the GPU is being used, we can try installing Docling in a virtual env.

sudo snap install astral-uv

Create the venv.

uv venv
source .venv/bin/activate

install docling 

uv pip install docling

Then make sure torch has rocm support

## Ollama

To use Ollama with ROCm acceleration you need to build both llama.cpp and ollama locally as there is still no native default support (at the time of writing).
