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

### Set up folder structure

Do not clutter your system folders. Keep everything inside your home directory or a dedicated /data folder. Since we are keeping it simple, let's use your home folder.

```sh
$ mkdir -p ~/server-data/{torrents,media/{movies,shows}}

# sudo apt install tree
$ tree ~/server-data
/home/serveradmin/server-data/
├── torrents/       <-- qBittorrent dumps files here
└── media/          <-- Jellyfin watches this
    ├── movies/     <-- You move clean movie files here
    └── shows/      <-- You move clean TV show files here
```

### Set up file manager

Install Midnight Commander:

```sh
sudo apt install -y mc
```

Install helper drivers for exFAT and NTFS drives:

```sh
sudo apt install -y exfat-fuse ntfs-3g
```

Add "Mount" option to F2 menu in Midnight Commander:

1. Open `mc`.
2. Press **F9** (Menu) > **Command** > **Edit menu file** > **User**.
3. Using `nano`, add these lines to the bottom of the file (CTRL + V):

```toml
+ t t
m       Mount USB (udisks)
        read -p "Partition (e.g. sdb1): " part
        udisksctl mount -b /dev/$part

u       Unmount USB (udisks)
        read -p "Partition (e.g. sdb1): " part
        udisksctl unmount -b /dev/$part
```

4. Save and exit using CTRL + O, Enter, CTRL + X.

When you are in `mc`, press **F2** and press **m**. It will ask you for the partition (like `sdb1`) and mount it for you instantly.

## Set up Tailscale

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

## Set up Docker

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

Add qBittorrent to your `docker-compose.yml`. It has a great Web UI that allows you to upload .torrent files or paste "Magnet Links" from your phone/tablet.

```yml
services:
  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    environment:
      - PUID=1000   # Critical: Ensures it has permission to write to your folders
      - PGUID=1000
      - TZ=Etc/UTC  # Change to your timezone if you want (e.g., Europe/Prague)
      - WEBUI_PORT=8080
    volumes:
      - ./qbittorrent_config:/config
      - ~/server-data/torrents:/downloads  # Maps the "torrents" folder
    ports:
      - "100.x.y.z:8080:8080" # Web UI (Tailscale only)
      - "6881:6881"           # Torrent traffic (TCP) - Needs to be public!
      - "6881:6881/udp"       # Torrent traffic (UDP) - Needs to be public!
    restart: unless-stopped

  jellyfin:
    image: jellyfin/jellyfin
    container_name: jellyfin
    network_mode: "host"
    volumes:
      - ./jellyfin_config:/config
      - ./jellyfin_cache:/cache
      # UPDATE THIS LINE TO MATCH NEW STRUCTURE:
      - ~/server-data/media:/media
    restart: 'unless-stopped'
```

Apply the changes:

```sh
cd ~/my-server
docker compose up -d
```

To change qBitorrent credentials, check current `admin` password from `docker log qbittorrent` and then in qBittorrent UI go to Tools > Options > Web UI > Authentication.

## Adjust firewall

```sh
# Allow everything from Tailscale
sudo ufw allow in on tailscale0
# Keep SSH open, just in case
sudo ufw allow ssh
# Allow any traffic coming in on the lo interface. This covers IPv4 (127.0.0.1) and IPv6 (::1).
sudo ufw allow in on lo
# Turn on firewall
sudo ufw enable
# Check enabled rules
sudo ufw status
```
