# ASUS ROG Flow Z13 (2025) - Omarchy Linux + CachyOS Kernel Guide

![Device](https://img.shields.io/badge/Device-ASUS_GZ302_(2025)-blue)
![CPU](https://img.shields.io/badge/CPU-AMD_Strix_Halo-red)
![Kernel](https://img.shields.io/badge/Kernel-CachyOS_Optimized-brightgreen)
![OS](https://img.shields.io/badge/OS-Omarchy_(Arch)-1793d1)

## Overview
This guide documents the ultimate setup for the **ASUS ROG Flow Z13 (2025 Model)** running **Omarchy Linux**.

Because this device features the **AMD Ryzen AI Max (Strix Halo)** architecture, a standard Arch install is not enough. This guide implements the **CachyOS Kernel** to enable:
*   **AVX-512 & NPU Support:** Hardware acceleration for AI tasks.
*   **Sched-Ext (scx_lavd):** Eliminates micro-stutters caused by the Strix Halo chiplet design.
*   **AMD P-State EPP:** Proper battery life and boost behavior.

---

## Prerequisites
1.  **BIOS Settings:** Reboot (`F2`) -> **Advanced** -> **Security** -> **Secure Boot** -> **Disable**.
2.  **Clean Install:** Assumes a fresh installation of **Omarchy Linux**.
3.  **Internet:** Required during setup.

---

## Phase 0: The Golden Rule (Update First)
**Do not skip this.** Installing custom kernels on an outdated base will break the system.

1.  Connect to Wi-Fi.
2.  Update system keyrings and packages:
    ```bash
    sudo pacman -Sy archlinux-keyring
    sudo pacman -Syu
    ```
3.  **Reboot.**

---

## Phase 1: The Engine (Kernel & Drivers)
We replace the generic kernel with **CachyOS** and install the ASUS control stack.

### 1. Install CachyOS Repositories

Install the CachyOS Repo Helper
```
curl https://mirror.cachyos.org/cachyos-repo.tar.xz -o cachyos-repo.tar.xz
tar xvf cachyos-repo.tar.xz
cd cachyos-repo
sudo ./cachyos-repo.sh
cd ..
```  
### 2. Fix Hyprland Dependencies (Critical)

Adding CachyOS updates system libraries that conflict with the stock Hyprland. We must reinstall Hyprland to match the new libraries.
1. Force remove the old conflicting Hyprland binary
    ```sudo pacman -Rdd hyprland```

2. Update the system (Type 'Y' to all replacements)
    ```sudo pacman -Syu```

3. Reinstall Hyprland (now from CachyOS)
    ```sudo pacman -S hyprland```

4. Install Nano (text editor)
    ```sudo pacman -S nano```

### 3. Install CachyOS Kernel

The Limine bootloader will automatically detect this kernel after installation.  
Install Kernel & Headers
```sudo pacman -S linux-cachyos linux-cachyos-headers```

### 4. Install ASUS Tools

Add G14 Repo
```sudo bash -c 'cat <<EOF >> /etc/pacman.conf

[g14]
Server = https://arch.asus-linux.org
EOF'
```

Import Keys & Install
```
sudo pacman-key --recv-keys 8F654886F17D497FEFE3DB448B15A6B0E9A3FA35
sudo pacman-key --lsign-key 8F654886F17D497FEFE3DB448B15A6B0E9A3FA35
sudo pacman -Sy asusctl rog-control-center
```
  
---

## Phase 2: The `asusd` Service Fix
The daemon may fail to enable out of the box on minimal installs due to a missing install section. Fix it manually:

1.  Edit the service file:
    ```bash
    sudo nano /usr/lib/systemd/system/asusd.service
    ```
2.  Add this block to the very bottom of the file:
    ```ini
    [Install]
    WantedBy=multi-user.target
    ```
3.  Reload and Enable:
    ```bash
    sudo systemctl daemon-reload
    sudo systemctl enable --now asusd
    ```

---

## Phase 3: Hardware Support (Firmware & Tablet)

1. Bleeding-Edge Firmware
  Required for 2025 Audio and NPU support.
    ```bash  
    yay -S linux-firmware-git
    ```
   _Critical: If prompted :: linux-firmware-git and linux-firmware are in conflict, type y (Yes) to remove the old version._
   
2. Tablet Utilities
  Installs rotation, virtual keyboard, and menu tools.
    ```bash    
    yay -S iio-hyprland-git wvkbd-mobintl wofi
    ```
3. Wi-Fi Stability Fix
   Prevents the MediaTek MT7925 card from disconnecting during sleep.
    ```bash
    echo "options mt7925e disable_aspm=1" | sudo tee /etc/modprobe.d/mt7925e.conf
    ```
---

## Phase 4: Hyprland Configuration
Add the following to your `~/.config/hypr/hyprland.conf`:

  ```ini
  # --- ASUS ROG Flow Z13 (2025) Hardware ---
  # High-DPI Scaling (2.0 is crisp, 1.6 for more space)
  monitor = eDP-1, preferred, auto, 2
  
  # Auto-Rotation
  exec-once = iio-hyprland
  
  # Tablet Input Mapping
  input {
      tablet { transform = 0; output = eDP-1; }
  }
  
  # --- Keybinds ---
  # Virtual Keyboard (Super + V)
  bind = $mainMod, V, exec, pkill wvkbd-mobintl || wvkbd-mobintl -L 300
  
  # ROG Power Menu (Dedicated Side Button / Super + A)
  # Note: Use 'wev' to verify side button code if XF86Launch1 doesn't work
  bind = , XF86Launch1, exec, ~/rog-quick.sh
  bind = $mainMod, A, exec, ~/rog-quick.sh
  
  # Keyboard Backlight
  bind = , XF86KbdBrightnessDown, exec, asusctl kbd_brightness -d
  bind = , XF86KbdBrightnessUp, exec, asusctl kbd_brightness -i
  ```
---

## Phase 5: The "ROG Control" Menu
Create a visual menu to switch power profiles without the terminal.

1.  Create `~/rog-quick.sh`:
    ```bash
    #!/bin/bash
    options="🔇 Silent\n⚖️ Balanced\n🚀 Performance\n⚡ Turbo 95W\n🔋 Limit 80%\n🔌 Limit 100%"
    selected=$(echo -e "$options" | wofi --dmenu --prompt "ROG Control")
    case $selected in
      "🔇 Silent") asusctl profile -P Quiet && notify-send "ROG" "Silent Mode" ;;
      "⚖️ Balanced") asusctl profile -P Balanced && notify-send "ROG" "Balanced Mode" ;;
      "🚀 Performance") asusctl profile -P Performance && notify-send "ROG" "Performance Mode" ;;
      "⚡ Turbo 95W") 
          asusctl profile -P Performance
          asusctl power-limits -s 95000 
          notify-send "ROG" "Turbo 95W Active" ;;
      "🔋 Limit 80%") asusctl -c 80 && notify-send "Battery" "Capped at 80%" ;;
      "🔌 Limit 100%") asusctl -c 100 && notify-send "Battery" "Uncapped" ;;
    esac
    ```
2.  Make executable: `chmod +x ~/rog-quick.sh`
3.  **Waybar Integration:** Add this module to `~/.config/waybar/config`:
    ```json
    "custom/asus": {
        "format": "⚡ {}",
        "exec": "asusctl profile -p | sed 's/Active profile is //'",
        "interval": 5,
        "on-click": "~/rog-quick.sh"
    }
    ```

---

## Phase 6: Final Verification
Reboot and select **CachyOS** from the boot menu.

| Component | Command / Action | Expected Result |
| :--- | :--- | :--- |
| **Kernel** | `uname -r` | Contains `cachyos` |
| **CPU Driver** | `cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_driver` | `amd-pstate-epp` |
| **Fans** | `asusctl -s` | Shows RPM and Temps |
| **Power** | `watch sensors` | Check `PPT` wattage changes with profiles |
| **Side Button** | Press Side Button | ROG Menu opens |
| **Rotation** | Rotate Device | Screen flips automatically |

---

## Extra Credit: Advanced Controls

### A. Advanced Gaming (HHD)
  If you use the Z13 as a handheld console (gyro/controller support):
  ```bash
  yay -S hhd-git
  sudo systemctl enable --now hhd.service
  ```
  Run hhd-gui to configure gyro and rumble.
  
### B. The "Turbo" Mode (Manual TDP)

The script in Phase 5 includes a "Turbo 95W" option.

  This uses asusctl power-limits -s 95000 to override the default 80W cap of the Performance profile.

  To Reset: Select a different profile or run asusctl power-limits --restore.

### C. Hardware Notes

  Side Shortcut Button: Mapped to XF86Launch1 (usually). Perfect for triggering the Power Menu or Virtual Keyboard.

  USB-C Charging: Third-party chargers work (65W-100W) but ASUS firmware may lock out the highest "Turbo" TDP modes. Use the official brick for heavy gaming.

---

  ## Troubleshooting

*   **No Audio:**
    *   Ensure `linux-firmware-git` is installed.
    *   Open `pavucontrol` (PulseAudio Volume Control) and ensure the correct output device (Smart Amp) is selected, not HDMI.
*   **Wi-Fi Drops:**
    *   Verify the config file exists: `cat /etc/modprobe.d/mt7925e.conf`
    *   It must contain `options mt7925e disable_aspm=1`.
*   **Boot Loop / Black Screen:**
    *   Double-check that **Secure Boot** is **DISABLED** in the BIOS (`F2` on boot). The CachyOS kernel is not signed for Secure Boot.

---

## Credits & Resources
*   **[ASUS-Linux.org](https://asus-linux.org):** The developers behind `asusctl` and the reverse-engineering efforts for ROG laptops.
*   **[CachyOS](https://cachyos.org):** For the optimized kernel and `sched-ext` scheduler work that makes Strix Halo usable.
*   **[Omarchy](https://omarchy.org):** For the lightweight, opinionated Arch base.


---

## Disclaimer
I am not responsible for bricked devices, dead SD cards, thermonuclear war, or you getting fired because the alarm app failed. 
*   **Voltage/TDP:** Modifying power limits (`asusctl power-limits`) can push hardware beyond factory specs. Use "Turbo" modes at your own risk.
*   **Secure Boot:** Disabling Secure Boot is required but may affect some Windows-only games if you dual-boot.

---

## License
This project is licensed under the MIT License - feel free to use, modify, and distribute.
