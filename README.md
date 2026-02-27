# power-saving-tweaks-for-linux
My personal power saving tweaks on linux, mainly laptops including desktops

i've been messing around with this stuff for a while now and figured i'd write it all down so i dont forget. most of this is tested on fedora KDE but a lot of it should work on other distros too. no guarantees tho lol

---

## Table of Contents
- [the basics first](#the-basics-first)
- [CPU stuff](#cpu-stuff)
- [display and GPU](#display-and-gpu)
- [KDE specific tweaks](#kde-specific-tweaks)
- [background services (kill the bloat)](#background-services-kill-the-bloat)
- [network and bluetooth](#network-and-bluetooth)
- [kernel parameters (advanced)](#kernel-parameters-advanced)
- [powertop](#powertop)
- [monitoring temps](#monitoring-temps)

---

## the basics first

before doing anything fancy make sure you have these installed:

```bash
sudo dnf install thermald lm_sensors powertop cpupower
sudo sensors-detect   # just hit enter for everything
sudo systemctl enable --now thermald
```

`thermald` is honestly one of the best things you can install, it actively manages thermals in the background and you kinda just forget about it

---

## CPU stuff

### power profiles daemon

fedora comes with `power-profiles-daemon` already. you can switch profiles from KDE's system settings under Power Management, or from terminal:

```bash
# check current profile
powerprofilesctl get

# set to power saver
powerprofilesctl set power-saver

# other options: balanced, performance
```

### auto-cpufreq (better option imo)

if you want something more dynamic than power-profiles-daemon, i'd recommend auto-cpufreq instead. it does a better job scaling the CPU based on actual load

```bash
sudo dnf install auto-cpufreq
sudo auto-cpufreq --install
```

after install it runs as a systemd service automatically. you can check what its doing with:

```bash
sudo auto-cpufreq --stats
```

> **note:** dont run both auto-cpufreq and power-profiles-daemon at the same time, they'll fight each other. pick one

### manually setting governor

if you want to just do it yourself:

```bash
# set all cores to powersave
sudo cpupower frequency-set -g powersave

# check current governor
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
```

---

## display and GPU

- lower screen brightness, seriously this has a huge impact on battery
- in KDE go to System Settings → Power Management and set screen to turn off after like 2-3 mins
- disable compositor effects or at least reduce them: System Settings → Display & Monitor → Compositor
  - switch to OpenGL 2.0
  - turn off "Allow applications to block compositing"

### AMD GPU

AMD handles power management pretty well on its own. you can verify with:

```bash
sudo dnf install radeontop
radeontop
```

### Nvidia GPU

nvidia is more annoying. open `nvidia-settings` and under PowerMizer set "Preferred Mode" to "Adaptive" or "Maximum Battery"

for newer nvidia drivers you can also do:

```bash
sudo nvidia-smi --persistence-mode=0
```

---

## KDE specific tweaks

KDE is pretty heavy out of the box, there's a bunch of stuff you can turn off

### desktop effects

System Settings → Workspace Behavior → Desktop Effects

turn these off or it'll chew through resources for no reason:
- Blur
- Background Contrast  
- Wobbly Windows (if you have it on)
- any transparency effects you dont actually use

### animations

System Settings → Workspace Behavior → General Behavior → Animation speed

drag it all the way to "Instant" if you want, or just make it faster. makes everything snappier too

### baloo file indexer

baloo runs in the background indexing your files for search. if you dont use KDE's file search much just turn it off

System Settings → Search → File Search → uncheck "Enable File Search"

or from terminal:

```bash
balooctl disable
balooctl purge   # cleans up the index files too
```

### akonadi

if you dont use KDE PIM stuff (Kontact, KMail, KOrganizer), akonadi is just sitting there using resources

```bash
akonadictl stop

# to stop it autostarting
mkdir -p ~/.config/akonadi
echo "[BasicPreferences]
DisableAutostart=true" >> ~/.config/akonadi/akonadiserverrc
```

---

## background services (kill the bloat)

check whats running and disable what you dont need:

```bash
# see user services
systemctl --user list-units --state=running

# see system services
systemctl list-units --state=running

# disable something (example)
sudo systemctl disable --now bluetooth  # if you never use bluetooth
```

also check KDE autostart: System Settings → Startup and Shutdown → Autostart

remove anything you dont recognize or dont need at login

---

## network and bluetooth

### wifi power saving

```bash
# check current state
iw dev wlan0 get power_save

# enable power saving
sudo iw dev wlan0 set power_save on
```

to make it permanent, create a udev rule:

```bash
sudo nano /etc/udev/rules.d/81-wifi-powersave.rules
```

add this line:
```
ACTION=="add", SUBSYSTEM=="net", KERNEL=="wlan*", RUN+="/sbin/iw dev %k set power_save on"
```

### bluetooth

if you dont use bluetooth just disable it entirely:

```bash
sudo systemctl disable --now bluetooth
rfkill block bluetooth
```

---

## kernel parameters (advanced)

> **warning:** some of these can cause instability depending on your hardware. test one at a time and see how it goes. if something breaks, boot with the old cmdline from grub and remove the parameter

edit your grub config:

```bash
sudo nano /etc/default/grub
```

find the line that says `GRUB_CMDLINE_LINUX` and add parameters inside the quotes. then update grub:

```bash
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

### parameters i use

```
GRUB_CMDLINE_LINUX="... quiet splash nmi_watchdog=0 pcie_aspm=force pcie_aspm.policy=powersupersave nvme.noacpi=1 snd_hda_intel.power_save=1 snd_hda_intel.power_save_controller=y"
```

breakdown of what each one does:

| parameter | what it does |
|-----------|-------------|
| `nmi_watchdog=0` | disables the NMI watchdog, saves a tiny bit, no real downside on a laptop |
| `pcie_aspm=force` | forces Active State Power Management on PCIe devices |
| `pcie_aspm.policy=powersupersave` | more aggressive ASPM policy |
| `nvme.noacpi=1` | disables ACPI for NVMe, can reduce temps on some systems |
| `snd_hda_intel.power_save=1` | puts audio card to sleep after 1 second of no use |
| `snd_hda_intel.power_save_controller=Y` | also powers down the audio controller |

### CPU vulnerability mitigations (controversial)

some people disable CPU mitigations (spectre, meltdown etc) to get performance/power back. i'm not recommending this for most people because security, but if you're on an isolated machine or just want to experiment:

```
mitigations=off
```

use your own judgment on this one

### intel specific

if you're on intel, also consider adding:

```
intel_idle.max_cstate=9
intel_pstate=active
```

### AMD specific

for AMD laptops:

```
amd_pstate=active
```

this enables the AMD P-State driver which is way better than the generic acpi-cpufreq. check if its already active first:

```bash
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_driver
```

if it says `amd-pstate-epp` youre already good

---

## powertop

powertop is great for seeing exactly whats waking up your CPU and draining power

```bash
sudo powertop
```

it shows you a breakdown by process and device. the "Tune" tab shows toggles you can enable

### auto-tune on boot

you can make powertop apply all its suggestions automatically at boot:

```bash
sudo nano /etc/systemd/system/powertop.service
```

paste this:

```ini
[Unit]
Description=Powertop auto-tune
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/sbin/powertop --auto-tune
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

then enable it:

```bash
sudo systemctl enable --now powertop.service
```

---

## monitoring temps

```bash
# install and set up sensors
sudo dnf install lm_sensors
sudo sensors-detect  # run through the wizard

# check temps
sensors

# watch temps live
watch -n 1 sensors
```

for a nicer TUI:

```bash
sudo dnf install htop
htop  # not exactly temps but shows CPU load well
```

for GPU temps on AMD:

```bash
cat /sys/class/hwmon/hwmon*/temp1_input  # divided by 1000 = celsius
# or just use radeontop
```

---

## quick checklist (priority order)

if you dont want to do everything, at least do these in order, biggest impact first:

- [ ] install and enable `thermald`
- [ ] set power profile to power-saver or install auto-cpufreq
- [ ] lower screen brightness
- [ ] disable Baloo if you dont use file search
- [ ] disable Akonadi if you dont use KDE PIM
- [ ] turn off KDE blur/transparency effects
- [ ] enable wifi power saving
- [ ] run `powertop --auto-tune` on boot
- [ ] add kernel parameters (test carefully)

---

feel free to open an issue or PR if something doesnt work for you or if i got something wrong. this is just what worked for me on my setup
