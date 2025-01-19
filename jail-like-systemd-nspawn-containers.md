# Host setup

## Setup a network bridge `br0`

```
sudo nmcli connection add type bridge con-name br0 ifname br0 && \
sudo nmcli connection add type bridge-slave con-name br0-slave ifname eth0 master br0 && \
sudo nmcli connection modify br0 ipv4.method auto && \
sudo nmcli connection up br0
```

Then reboot

# Container setup

## Create container FS

```
sudo debootstrap stable /var/lib/machines/<container-name> http://deb.debian.org/debian
```

Change the hostname by editing

```
/var/lib/machines/<container-name>/etc/hostname
```

## Set root password in container

First start container using

```
sudo systemd-nspawn -D /var/lib/machines/<container-name>
```

Then use `passwd` to set new root password.
Finally type `exit` to shut the container down.

## Boot container

To properly boot the container use
```
sudo systemd-nspawn --boot --network-bridge=br0 -D /var/lib/machines/<container-name>
```

## Setup DHCP

In the container create a new systemd service file

```
nano /etc/systemd/system/dhcp.service
```

This is the content of the service file:

```
[Unit]
Description=Start DHCP client on host0 interface
After=network.target

[Service]
ExecStart=/usr/sbin/dhclient host0
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Enable the service via

```
systemctl enable dhcp.service
```

Test it by shuting down the container (`shutdown now`) and reboot it with the command from above.

## Setup non-root user

First install `sudo`.

```
apt install sudo
```

Create a new user (assuming zsh is already installed).

```
useradd -m -s /usr/bin/zsh <user>
```

Add the user to the sudo group.

```
usermod -aG sudo <user>
```

Log out via `logout`, try to login with the new user and try to execute a sudo command. This way you can be sure that you don't lock yourselve out during the next step.

Now deactivate login via root. Edit `/etc/passwd`, find the line for root (probably the first one) and change the shell to `/usr/sbin/nologin`:

```
root:x:0:0:root:/root:/usr/sbin/nologin
```

Logout and try to login as root. Should not be possible now.

## Automatically start the container during boot as systemd service

When starting the container as service you will rely on `machinectl` to open a shell in the container. Therefore dbus is needed in the container.

```
sudo apt install dbus
```

Create a new service file on the host.

```
/etc/systemd/system/<container-name>-container.service
```

This is the content of the file.

```
[Unit]
Description=<container-name> container
After=network.target

[Service]
ExecStart=/usr/bin/systemd-nspawn --boot --network-bridge=br0 -D /var/lib/machines/<container-name>
Restart=always
User=root

[Install]
WantedBy=multi-user.target
```

Enable the service and start it.

```
sudo systemctl enable <container-name>-container.service
sudo systemctl start <container-name>-container.service
```

You can now login via machinectl.

```
machinectl shell <user>@<container-name>
```
