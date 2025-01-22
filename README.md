# Jail like Linux containers on Raspberry Pi OS

When I bought my new Raspberry Pi 5 I wanted to have something similar like Jails on my FreeBSD server. I did a little research and ended up using `systemd-nspawn` containers on Raspberry Pi OS. To make creating new containers easier in the future, I implemented the container setup in an Ansible playbook `build_container.yaml`.

# Host setup

## Setup a network bridge `br0`

As a pre-requisite you have to set up a network bridge on Raspberry Pi OS. My choice was to have the container interfaces bridged with the host interface. The containers will then appear as separate hosts on the network and trying to obtain an IP address via DHCP as the Pi does itself. The container playbook assumes that there is a network bridge called `br0`.

```
sudo nmcli connection add type bridge con-name br0 ifname br0 && \
sudo nmcli connection add type bridge-slave con-name br0-slave ifname eth0 master br0 && \
sudo nmcli connection modify br0 ipv4.method auto && \
sudo nmcli connection up br0
```

Then reboot.

Note: If you are on a SSH connection, be sure to execute all the commands at once, like in the listing. Otherwise you will break your SSH connection in the process. Ask me how I know...

# Container setup

This part is implemented in the `build_container.yaml` Ansible playbook. Install Ansible and execute the playbook. 

The playbook sets up a Debian root file system with the current Debian stable release. Change the `debian_suite` variable in the playbook if you want a different release.

The playbook requires these variables to be set (e.g. via `vars` in the the inventory):

* `container_name`: The name of the container. This will also be the hostname on the network.
* `container_user`: The playbook sets up a non-root user in the container. The playbook will prompt you for a password for the user when you play it.

After you container is set up, it boots automatically after you boot your Pi. It uses a systemd service called `<container_name>-container.service` for that.

You can list all running containers with `machinectl list`. You can login to a container via `machinectl shell <container_user>@<container_name>`.

## Example usage

I define a basic inventory with only my Raspberry Pi in it:

**inventory.yaml**

```yaml
raspis:
  hosts:
    container_host:
      ansible_host: myraspi
      container_name: grafana
      container_user: batman
```

Then I execute the playbook with

```bash
ansible-playbook build_container.yaml -i inventory.yaml --ask-become-pass
```