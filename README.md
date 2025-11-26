# selfhosted-server-config

## Install Ubuntu Server

Use Ubuntu Server LTS ISO, use entire disk during install, do not install any snaps or openSSH package.

```sh
# Update all system packages 
sudo apt upgrade
# Install OpenSSH
sudo apt install -y openssh-server
# Ensure SSH server starts on boot
sudo systemctl enable --now ssh
# Find your IP address
ip a

# Look for the line starting with inet under your ethernet interface (usually named enp... or eth0). It will look like 192.168.1.XX.
```

Now, continue working on another laptop, keeping the server machine alone. On a remote machine, open terminal and log in remotely by using `ssh adminuser@192.168.1.XX`.

### Disable screen after inactivity

Edit GRUB config file:

```sh
sudo nano /etc/default/grub
```

Find the line starting with `GRUB_CMDLINE_LINUX_DEFAULT`.

Add `consoleblank=60` inside the quotes. Change it to look like this:

```sh
# This tells the kernel to "Turn off the video signal if no one types for 60 seconds"
GRUB_CMDLINE_LINUX_DEFAULT="consoleblank=60"
```

Save and exit by pressing CTRL + 0, Enter, CTRL + X.

Update GRUB and apply:

```sh
sudo update-grub
sudo reboot
```

### Disable sleep by closed lid

To stop the lid closing from triggering sleep of the server, type this command to edit the config:

```sh
sudo nano /etc/systemd/logind.conf
```

* Scroll down to find `#HandleLidSwitch=suspend`.
* Change it to: `HandleLidSwitch=ignore` (Make sure to delete the `#`).
* Press Ctrl+O, Enter (to save), Ctrl+X (to exit).

Apply anf turn off screen:

```sh
sudo systemctl restart systemd-logind
```

### Expand logical volume

Ubuntu Serveer uses LVM (Logical Volume Manager). By default, it reserves half the disk space "just in case" you want to create a separate partition later. Since you are hosting a simple media server, you don't need this reservation.

```sh
# Check current state, look for /
df -h

# Expand the logical volume (LV)
# This tells LVM: "Take the ubuntu-lv volume and add 100% of the remaining free space to it".
sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv

# Resize the filesystem
# This tells the actual filesystem: "The container you live in just got bigger, please expand to fill the room".
sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv

# Verify size
# / should now show a different size
df -h
```

## Tailscale

```sh
# Install Tailscale
curl -fsSL https://tailscale.com/install.sh | sh

# Start Tailscale service
# It prints a URL. Copy/Paste that into your browser to authenticate.
sudo tailscale up

# Get secure IP and write it down for docker compose
tailscale ip -4
```

In the Tailscale Admin Console in your browser, for the new server machine, disable ke expiry. If not done, the server disconnects from Tailscale in 90 days.

## Docker

```sh
# Install Docker using automatic installer script
curl -fsSL https://get.docker.com | sudo sh

# By default, you have to type sudo before every docker command. Let's fix that so your user can run docker directly.
sudo usermod -aG docker $USER

# Log out and log back in for the permission change to take effect.
exit

# Verify everything works
docker compose version
```

Create `docker-compose.yml`:

```sh
mkdir my-server
cd my-server
nano docker-compose.yml
```

```yml
services:
  immich-server:
    image: ghcr.io/immich-app/immich-server:release
    # ... (standard immich config lines) ...
    ports:
      # BINDING TO TAILSCALE IP ONLY
      - "100.x.y.z:2283:3001" 
      # Format is "IP:HostPort:ContainerPort"
      # If you just put "2283:3001", it exposes it to your whole LAN. 
      # By adding the IP, ONLY Tailscale devices can see it.

  jellyfin:
    image: jellyfin/jellyfin
    network_mode: host # Jellyfin often needs host mode for DLNA/casting.
    # If using host mode, you must use UFW firewall (see below) 
    # because host mode ignores port bindings.
```

## Adjust firewall

```sh
# Allow EVERYTHING from Tailscale
sudo ufw allow in on tailscale0
# Keep SSH open just in case
sudo ufw allow ssh
# Turn firewall on. This effectively blocks external access to Jellyfin unless it's coming through the Tailscale tunnel
sudo ufw enable
```
