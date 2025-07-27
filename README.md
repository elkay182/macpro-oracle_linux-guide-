# Oracle Linux 9.6 on the Mac Pro 6,1 (“Trash-Can”)
*A proven walk-through that mirrors the exact steps we used to get a clean, bootable system.*

---

## 📑 Table of Contents
1. [Why This Is Difficult](#-why-this-is-difficult)  
2. [Prerequisites](#-prerequisites)  
3. [Installation — the **No-Boot-Loader** method](#-installation—the-no-boot-loader-method)  
4. [Post-Install Tasks](#-post-install-tasks)  
5. [Troubleshooting FAQ](#-troubleshooting-faq)  
6. [Outcome & Next Steps](#-outcome--next-steps)  
7. [References](#-references)

---

## 🔍 Why This Is Difficult

| Quirk | Impact |
|-------|--------|
| **Apple “macEFI” GUID** | Anaconda can’t re-format the proprietary Apple EFI slice → *“resource to create format ‘macefi’ unavailable”*. |
| **mactel-boot package missing** | The installer throws an error unless you disable boot-loader installation. |
| **FirePro D700 (Tahiti) stalls** | The legacy `radeon` driver can hang early boot; we use `nomodeset` during install and later switch to `amdgpu`. |
| **Wi-Fi (BCM4360)** | Needs proprietary **wl** driver — best compiled later. |
| **Headless installs** | Text mode avoids GUI lock-ups and allows rescue via console. |

---

## 🛠️ Prerequisites

| Item | Notes |
|------|-------|
| **Oracle Linux 9.6 Boot ISO** | `OracleLinux-R9-U6-x86_64-boot.iso` |
| **USB (≥16 GB)** | Flash with `dd`:<br>`sudo dd if=OracleLinux*.iso of=/dev/sdX bs=4M status=progress && sync` |
| **Ethernet cable** | Wired first; Wi-Fi later. |
| **Keyboard & monitor** | Needed for the Mac Pro during install. |
| **Firmware prep** | Startup Security Utility → *Allow booting from external media*. |

---

## 🚀 Installation — the **No-Boot-Loader** method

_These are the exact steps we confirmed work (adapted from Bugzilla Comment 16)._

> **Big picture**: Boot text-mode → tell the installer **not** to install a boot loader → manually create a fresh 600 MB EFI slice → finish install → rescue shell → `grub2-mkconfig`.

### 1 Boot the text installer

```text
⌥  →  EFI Boot → highlight “Install Oracle Linux” →  e
linuxefi … append:
   inst.text nomodeset radeon.modeset=0
Ctrl-X

2 Normal install-hub tasks

Set network, locale, packages as usual.

3 Disk partitioning — turn off boot loader
	1.	Installation Destination → select your SSD.
	2.	At the bottom click “Full disk summary and boot loader”.
	•	Select the disk.
	•	Click Do not install boot loader → Close.
	3.	Back on Storage Configuration, choose Custom → Done.

4 Manual partition layout
	1.	Delete old Mac partitions if present.
	2.	Click + to add partitions:

Mount Point	Size	Device Type	File System
/boot/efi	600 MB	Standard Partition	EFI System Partition
/boot	1 GB	Standard Partition	ext4
/	(rest)	LVM (default)	xfs
swap	4–8 GB	LVM	swap

Tip: If you hit size limits (e.g., 500 GB disk) shrink /home or / a bit first, then add the new /boot/efi.

	3.	Done → Apply Changes.

5 Begin installation
	•	Ignore the red mactel-boot missing warning → click Yes/Ignore.
	•	Installation completes, reboots; you’ll land at a minimal grub> prompt (no grub.cfg yet) — that’s expected.

6 Rescue shell → generate grub.cfg
	1.	Boot the ISO again → Troubleshooting → Rescue a Red Hat Enterprise Linux system → option 1 (mount on /mnt/sysimage).
	2.	At the shell:

chroot /mnt/sysimage
# sanity-check:
ls -l /boot/ /boot/efi/EFI/redhat
# write config:
grub2-mkconfig -o /boot/efi/EFI/redhat/grub.cfg
exit
reboot

Ignore harmless reload ioctl messages.

7 First SELinux relabel & login
	•	After the SELinux relabel reboot, log in as your wheel user (GUI or console).

⸻

🔧 Post-Install Tasks

A. Switch to amdgpu (remove nomodeset)

sudo tee /etc/modprobe.d/amdgpu.conf >/dev/null <<'EOF'
options amdgpu si_support=1 cik_support=0
blacklist radeon
options radeon modeset=0
EOF

sudo sed -i 's/^GRUB_CMDLINE_LINUX=.*/GRUB_CMDLINE_LINUX="quiet rhgb amdgpu.si_support=1 radeon.modeset=0"/' /etc/default/grub
sudo grub2-mkconfig -o /boot/efi/EFI/redhat/grub.cfg
sudo dracut -f --kver "$(uname -r)"
sudo reboot

B. Lock root & harden SSH

sudo passwd -l root
sudo sed -i 's/^#PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sudo systemctl reload sshd

C. (Optional) Build Broadcom Wi-Fi driver

sudo dnf install -y dkms gcc make kernel-uek-devel-$(uname -r)
curl -L -o wl.tar.gz \
  https://www.broadcom.com/docs/linux_sta/hybrid-portsrc_x86_64-v5_100_82_112.tar.gz
tar xzf wl.tar.gz && cd hybrid-portsrc_x86_64-*
make && sudo make install
echo -e "blacklist b43\nblacklist bcma\nblacklist brcmsmac\nblacklist ssb" | \
  sudo tee /etc/modprobe.d/broadcom-blacklist.conf
sudo dracut -f --kver "$(uname -r)"
sudo reboot


⸻

❓ Troubleshooting FAQ

Symptom	Remedy
“failed to save storage configuration” loop	You forgot the Do not install boot loader toggle or didn’t wipe GPT.
Stuck at prohibitory icon	grub.cfg missing → rescue shell → grub2-mkconfig.
Black console after GRUB	Keep nomodeset until you switch to amdgpu.
Wi-Fi absent	Compile wl driver (see above) or stay on Ethernet.


⸻

🎯 Outcome & Next Steps
	•	Oracle Linux 9.6 boots natively via a standard EFI System Partition.
	•	FirePro D700 uses amdgpu (or safe nomodeset).
	•	Root logins disabled; wheel user via sudo.
	•	Ready for Podman / Docker, OpenWebUI, and remote GPU endpoints (Ollama on a 3090 gaming PC).

⸻

📚 References
	•	Red Hat Bug 1751311 – mactel-boot loop
https://bugzilla.redhat.com/show_bug.cgi?id=1751311#c16
	•	Broadcom STA driver
https://www.broadcom.com/support/download-search?pf=Wireless+LAN+Infrastructure
	•	Oracle Linux docs
https://docs.oracle.com/en/operating-systems/

⸻

License: MIT • Author: Your Name  • PRs welcome!

