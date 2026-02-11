## Advanced: Setting up an AMD Computer for Machine Learning

While NVIDIA is the common standard, AMD hardware has become a powerful alternative for ML research thanks to the **ROCm (Radeon Open Compute)** ecosystem. If you are building on AMD, your primary tool is the ROCm stack, which allows frameworks like PyTorch to interface with your GPU.

---

### Linux Instructions (Recommended)

For a fresh install, we recommend **Ubuntu 24.04** or **22.04** for the most stable driver support.

#### Step 1: Install ROCm 6.4 Drivers

If you are setting up from scratch, use the following commands to install the ROCm 6.4 stack on Ubuntu 24.04:

1. **Update the system:**
```bash
sudo apt update

```


2. **Download and install the AMDGPU installer:**
```bash
wget https://repo.radeon.com/amdgpu-install/6.4/ubuntu/noble/amdgpu-install_6.4.60400-1_all.deb
sudo apt install ./amdgpu-install_6.4.60400-1_all.deb
sudo apt update

```


3. **Install the ROCm stack:**
*Note: The `--no-dkms` flag is used here to simplify the installation if you are using the stock Ubuntu kernel.*
```bash
sudo amdgpu-install -y --usecase=rocm --no-dkms

```


4. **Configure User Permissions:**
You must give your user account permission to access the GPU hardware.
```bash
sudo usermod -a -G render $USER
sudo usermod -aG video $USER

```


5. **Refresh and Verify:**
Restart your terminal or run `source ~/.bashrc`. Then, verify the installation with:
```bash
rocm-smi

```


If successful, you will see a table listing your AMD GPU, temperature, and power usage.

---

### Windows Instructions (via WSL)

To use AMD hardware for ML on Windows, you must utilize **WSL (Windows Subsystem for Linux)**. This allows you to run a Linux-based ROCm environment on top of Windows.

#### Step 1: Install WSL

Open Windows PowerShell as an Administrator and run:

```powershell
wsl --install
wsl --set-default Ubuntu

```

#### Step 2: Install AMD Adrenalin Driver

For ROCm to bridge correctly between Windows and WSL, you must have the **AMD Adrenalin v25.3.1 driver** (or newer) installed on your host Windows system.

* Download this directly from the [AMD Drivers & Support page](https://www.amd.com/en/support).

#### Step 3: Install ROCm for WSL

Once the Windows driver is installed, enter your WSL terminal and follow the [Official ROCm for WSL Guide](https://www.google.com/search?q=https://rocm.docs.amd.com/en/latest/deploy/linux/wsl/wsl.html) to map the hardware correctly.

---

### Summary of AMD vs. NVIDIA Setup

| Feature | NVIDIA Setup | AMD Setup |
| --- | --- | --- |
| **Primary Library** | CUDA | ROCm |
| **Best Linux Distro** | Ubuntu 22.04 / Pop!_OS | Ubuntu 24.04 |
| **Verification Tool** | `nvidia-smi` | `rocm-smi` |
| **Ease of Setup** | High (Industry Standard) | Moderate (Improving rapidly) |

---
