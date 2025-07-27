# Oracle Linux 9.6 on the Mac Pro 6,1 (“Trash-Can”)
*A step-by-step field guide — EFI fixes, GPU work-arounds, and homelab tips*

<p align="center">
  <img src="docs/img/trashcan_macpro.jpg" width="320" alt="Mac Pro 6,1">
</p>

---

## 📑 Table of Contents
1. [Why This Is Tricky](#-why-this-is-tricky)  
2. [What You’ll Need](#-what-youll-need)  
3. [Step-by-Step Install (UEFI Text Mode)](#-step-by-step-install-uefi-text-mode)  
4. [Post-Install Fixes](#-post-install-fixes)  
5. [Troubleshooting FAQ](#-troubleshooting-faq)  
6. [Outcome & Next Steps](#-outcome--next-steps)  
7. [References & Links](#-references--links)

---

## 🔍 Why This Is Tricky

| Quirk | Impact |
|-------|--------|
| **Apple “macEFI” GUID** | Anaconda’s storage module can’t re-format Apple’s proprietary EFI slice → endless *“resource to create format ‘macefi’ unavailable”* loop. |
| **GRUB vs. Apple firmware** | Must wipe old GPT/APFS metadata **and** skip the `mactel-boot` step, then manually generate `grub.cfg`. |
| **FirePro D700 stalls** | Legacy **radeon** driver sometimes locks the console. Need `nomodeset` or force **amdgpu**. |
| **Broadcom Wi-Fi** | BCM4360 requires proprietary **wl** driver; must be compiled for UEK. |
| **Headless install** | Use **text-mode** to avoid GUI lock-ups and enable SSH early. |

---

## 🛠️ What You’ll Need

| Item | Details |
|------|---------|
| **Oracle Linux 9.6 ISO** | `OracleLinux-R9-U6-x86_64-boot.iso` |
| **≥16 GB USB** | Flash with `dd`:<br>`sudo dd if=OracleLinux*.iso of=/dev/sdX bs=4M status=progress && sync` |
| **Wired Ethernet** | Wi-Fi configured later. |
| **Keyboard + monitor** | Plugged into Mac Pro for the install. |
| **(Optional) WoL MAC** | Needed only if you’ll wake a remote GPU box. |

**Firmware checklist**

1. macOS Recovery ▶ **Startup Security Utility** ▶ *Allow booting from external media*.  
2. PCIe Wake-on-LAN **enabled** (if you plan WoL).  
3. No other changes to SIP or Secure Boot are required.

---

## 🚀 Step-by-Step Install (UEFI Text Mode)

> **TL;DR**  Wipe GPT → Automatic partitioning → Skip boot-loader → rescue shell → `grub2-mkconfig`.

### 1  Boot text installer

```text
⌥  →  EFI Boot → highlight “Install Oracle Linux” →  e
linuxefi … append:
   inst.text nomodeset radeon.modeset=0
Ctrl-X

2  Open shell → wipe Apple GPT

# Press “2 – Shell” in the text menu
lsblk -d                      # find SSD (usually /dev/sda)
dd if=/dev/zero of=/dev/sda bs=1M count=16
parted /dev/sda --script mklabel gpt
exit

3  Automatic partitioning

5 → Installation Destination
[X] /dev/sda   |   Automatic partitioning   |   Delete all on selected drives
c → w

Installer creates:

Device	Size	Mount	FS
/dev/sda1	200 MiB	/boot/efi	vfat
/dev/sda2	1 GiB	/boot	ext4
LVM VG ol	rest	/ (xfs) + swap	xfs

When mactel-boot missing appears, press Ignore.

4  Finalize installer
	•	Set root password
	•	Create wheel user (check “Make this user administrator”)
	•	b – Begin Installation ▪ wait ≈5 min.

5  Rescue GRUB (first reboot)
	1.	Mac reboots to grub> minimal shell → insert USB → choose Troubleshooting ▶ Rescue → option 1 (mount RW).
	2.	Generate config:

chroot /mnt/sysroot
grub2-mkconfig -o /boot/efi/EFI/redhat/grub.cfg
exit
reboot

Hold ⌥, pick Oracle Linux. GRUB menu appears ✅.

⸻

🔧 Post-Install Fixes

A. Switch D700 to amdgpu

sudo tee /etc/modprobe.d/amdgpu.conf <<'EOF'
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

C. (Optional) Build Broadcom Wi-Fi wl driver

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

Symptom	Fix
“failed to save storage configuration” loop	You skipped the wipe. Boot USB, run dd + parted mklabel gpt again.
Stuck at Apple prohibitory icon	grub.cfg missing. Rescue → grub2-mkconfig.
Black screen after GRUB	Keep nomodeset, then migrate to amdgpu.
Wi-Fi absent	Build proprietary wl driver as shown above.
Cannot SSH as root	That’s intended — use your wheel account + sudo.


⸻

🎯 Outcome & Next Steps
	•	Oracle Linux 9.6 boots natively on Mac Pro 6,1 with a standard UEFI GRUB.
	•	FirePro D700 runs on amdgpu (or safe nomodeset fallback).
	•	System ready for Podman / Docker, OpenWebUI, Ollama on a remote GPU box, etc.
	•	Optional Wake-on-LAN proxy lets a gaming PC with RTX 3090 spin up only when AI is requested.

⸻

📚 References & Links
	•	RH Bug 1751311 — mactel-boot installer loop
https://bugzilla.redhat.com/show_bug.cgi?id=1751311#c16
	•	Broadcom STA driver download
https://www.broadcom.com/support/download-search?pf=Wireless+LAN+Infrastructure
	•	Oracle Linux documentation
https://docs.oracle.com/en/operating-systems/
	•	AMD GPU Linux support matrix
https://rocm-documentation.readthedocs.io/

⸻

License & Authorship — MIT License • Contributions welcome • Created 2025

