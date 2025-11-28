# How I set up my local Ubuntu server

On a refurbished Dell ultrabook. And you can too.

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

## Mounting drives

```sh
# Enumerate drives
$ lsblk
NAME                      MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda                         8:0    0 238.5G  0 disk 
├─sda1                      8:1    0 238.4G  0 part 
└─sda2                      8:2    0    32M  0 part 
nvme0n1                   259:0    0 476.9G  0 disk 
├─nvme0n1p1               259:1    0     1G  0 part /boot/efi
├─nvme0n1p2               259:2    0     2G  0 part /boot
└─nvme0n1p3               259:3    0 473.9G  0 part 
  └─ubuntu--vg-ubuntu--lv 252:0    0 473.9G  0 lvm  /

# Mount a specific drive
$ udisksctl mount -b /dev/sdb1
# Unmount a specific drive
$ udisksctl unmount -b /dev/sdb1
Unmounted /dev/sda1.
```

## Set up logging

For a "user-friendly" view of network traffic, CPU usage, and disk activity, use Netdata. It installs with a single command. It creates a real-time dashboard accessible via your browser. Real-time upload/download speeds, which container is eating RAM, and if your CPU is throttling.

```sh
wget -O /tmp/netdata-kickstart.sh https://my-netdata.io/kickstart.sh && sh /tmp/netdata-kickstart.sh
```

## Set up file manager

Install Midnight Commander:

```sh
sudo apt install -y mc
```

Install helper drivers for exFAT and NTFS drives:

```sh
sudo apt install -y exfat-fuse ntfs-3g
```

## Set up VPN

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

## Set up firewall

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
# Define services in the docker-compose file
nano docker-compose.yml

$ cat docker-compose.yml
services:
  serviceA:
    # config
  serviceB:
    # config
```

Once services are configured, run `docker compose`:

```
cd ~/my-server
docker compose up -d
```

### qBitTorrent

```yml
  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    environment:
      - PUID=1000   # Critical: Ensures it has permission to write to your folders
      - PGUID=1000
      - TZ=Etc/UTC  # Change to your timezone if you want
      - WEBUI_PORT=8080
    volumes:
      - ./qbittorrent_config:/config
      - ~/server-data/torrents:/downloads  # Maps the "torrents" folder
    ports:
      - "<TailscaleServerIPv4>:8080:8080" # Web UI (Tailscale only)
      - "6881:6881"           # Torrent traffic (TCP) - Needs to be public!
      - "6881:6881/udp"       # Torrent traffic (UDP) - Needs to be public!
    restart: unless-stopped
```

#### Change user credentials

On start, qBittorrent generates a unique password for the `admin` user, which must be manually set. Otherwise, it's different on each start.

1. Run `docker log qbittorrent`.
2. Find the generated admin password in the logs.
3. In the qBittorrent web UI:
    1. Use the password to sign in as the `admin` user.
    2. In **Tools** > **Options** > **Web UI** > **Authentication**, change the username and password.
    3. **Save**.

The newly set password is stored in encrypted format in `qBittorrent.conf` in the `WebUI\Password_PBKDF2` key. If you forget it or mistype, stop qBittorrent using docker stop qbittorrent, delete the key from config, start the service, and then follow the steps again.

### Jellyfin

```yml
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

### Dozzle

View Docker container logs live in a single interface.

```yml
  dozzle:
    image: amir20/dozzle:latest
    container_name: dozzle
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    ports:
      - "100.x.y.z:8888:8080" # Accessible at port 8888
    restart: unless-stopped
```

### Immich

Photo media management
