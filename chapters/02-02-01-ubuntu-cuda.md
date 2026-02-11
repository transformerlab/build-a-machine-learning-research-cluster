# Setting Up Ubuntu 22.04 for ML: Step-by-Step

As mentioned, do not fear "breaking" this setup. If you encounter a dependency loop or a driver conflict that persists after a reboot, simply reformat the drive and start fresh. On a single-user workstation, a clean slate is often faster than deep-system troubleshooting.

---

### Phase 1: Ubuntu 22.04 Installation

1. **Download & Flash:** Download the [Ubuntu 22.04 LTS ISO](https://ubuntu.com/download/desktop) and flash it to a USB drive using a tool like **BalenaEtcher** or **Rufus**.
2. **BIOS Settings:** Ensure **Secure Boot** is **Disabled** in your BIOS. Secure Boot can block proprietary NVIDIA drivers from loading.
3. **The Install:** Boot from the USB.
* Choose **"Install Ubuntu."**
* **CRITICAL:** Under "Updates and other software," check the box: **"Install third-party software for graphics and Wi-Fi hardware."** This simplifies the initial driver handshake.
* Select "Erase disk and install Ubuntu" for a clean ML environment.



### Phase 2: System Preparation

Once logged in, open your terminal (**Ctrl+Alt+T**) and ensure your system is fully patched.

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install build-essential dkms -y

```

### Phase 3: Install NVIDIA Drivers

We recommend using the **PPA repository** for the most stable, up-to-date drivers.

1. Add the graphics PPA:
```bash
sudo add-apt-repository ppa:graphics-drivers/ppa
sudo apt update

```


2. Identify and install the recommended driver:
```bash
ubuntu-drivers devices

```


*Look for the version labeled `recommended` (e.g., nvidia-driver-535). Install it:*
```bash
sudo apt install nvidia-driver-535 -y

```


3. **Reboot your machine:**
```bash
sudo reboot

```


4. **Verify:** After reboot, run `nvidia-smi`. You should see a table displaying your GPU and driver version.

### Phase 4: Install CUDA Toolkit

Avoid the generic `apt install nvidia-cuda-toolkit` command as it often installs outdated versions. Use the official NVIDIA repository.

1. **Set up the repository pin:**
```bash
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-ubuntu2204.pin
sudo mv cuda-ubuntu2204.pin /etc/apt/preferences.d/cuda-repository-pin-600

```


2. **Fetch the repository keys:**
```bash
sudo apt-key adv --fetch-keys https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/3bf863cc.pub
sudo add-apt-repository "deb https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/ /"
sudo apt update

```


3. **Install CUDA:**
```bash
sudo apt install cuda -y

```



### Phase 5: Environment Configuration

You must tell Ubuntu where to find the CUDA binaries by editing your `.bashrc` file.

1. Open the file: `nano ~/.bashrc`
2. Add these lines to the very bottom:
```bash
export PATH=/usr/local/cuda/bin${PATH:+:${PATH}}
export LD_LIBRARY_PATH=/usr/local/cuda/lib64${LD_LIBRARY_PATH:+:${LD_LIBRARY_PATH}}

```


3. Save and exit (**Ctrl+O, Enter, Ctrl+X**), then refresh your terminal:
```bash
source ~/.bashrc

```



---

### Verification

Run these two commands to ensure your hardware and software are communicating:

* **`nvidia-smi`**: Confirms the driver is active and the GPU is visible.
* **`nvcc --version`**: Confirms the CUDA compiler is correctly in your system path.

