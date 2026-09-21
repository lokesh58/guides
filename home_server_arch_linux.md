# Home Server Arch Linux Guide

This guide is to be used as complement to the [Arch Install](./arch_install.md) guide.

## Console Blanking

Screen doesn't go blank by default in Arch Linux.
For a server laptop that's unnecessary power usage.
We can enable blanking using kernel parameters.

```bash
sudo nvim /etc/systemd/server.conf
```

```conf
# Blank the screen after 10 second timeout
consoleblank=10
```

Make sure to rebuild the kernel after change:

```bash
sudo mkinitcpio -P
```

**Important:** Ensure the backlight has gone off after the console blanking.
For some laptops the backlight may remain on, even when screen is blank.
(Use google/AI to find how to check this).

## Ignoring LidSwitch

Accidentally/deliberately closing the laptop lid should not power down the laptop.
This can be handled via systemd.

```bash
sudo mkdir /etc/systemd/logind.conf.d
sudo nvim /etc/systemd/logind.conf.d/server.conf
```

```conf
[Login]
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
```

## Setup auto power off on low battery

In case of power outage, the laptop should automatically power down if power isn't back for a long time.
The battery threshold can be updated based on battery condition, we should need at least 5 minutes for a proper shutdown.
Use upower for doing this:

```bash
sudo pacman -S upower
EDITOR=nvim sudoedit /etc/UPower/UPower.conf
```

```conf
NoPollBatteries=false # set to true if hardware sends events for battery percentage changes (does not happen in Dell G15)
IgnoreLid=true
UsePercentageForPolicy=true
PercentageLow=40.0
PercentageCritical=35.0
PercentageAction=30.0
CriticalPowerAction=PowerOff
```
