# VM preparation

All VMs / machines are **Debian 13**. The only exception is specific user
applications.

By standard, EFDI VMs are created with these parameters:

- 4 CPU
- 4 GB RAM
- 32 GB disk space

These parameters are a flat baseline — consider changing them for specific
applications.

## OS install

Some steps are quite important. Follow the manual fully.
To see a bigger image, right-click it and open in a new tab.

> [!IMPORTANT]
> Go through every screen and apply exactly the configuration shown in the
> images. Do not skip or fast-forward through the installer steps — the settings
> in the screenshots are required.

<img src="media/os_step1.png" alt="os_step1" width="45%">

Pick your location and locales.

<img src="media/os_step2.png" alt="os_step2" width="45%"> <img src="media/os_step3.png" alt="os_step3" width="45%">
<img src="media/os_step4.png" alt="os_step4" width="45%"> <img src="media/os_step5.png" alt="os_step5" width="45%">
<img src="media/os_step6.png" alt="os_step6" width="45%"> <img src="media/os_step7.png" alt="os_step7" width="45%">

> [!WARNING]
> If your network has DHCP configured and you want to enter static network
> configuration mode, click `<Cancel>` at the moment DHCP detection is happening.
> If you miss it and the installer configures the network via DHCP —
> you can set a static configuration later by editing the config files (see
> [Static networking](#2-static-networking-only-if-not-set-during-install) in the
> OS configuration section).

Configure the network.
IP and DNS belong to the network config. Keep the domain name empty.

<img src="media/os_step8.png" alt="os_step8" width="45%"> <img src="media/os_step9.png" alt="os_step9" width="45%">
<img src="media/os_step10.png" alt="os_step10" width="45%"> <img src="media/os_step11.png" alt="os_step11" width="45%">
<img src="media/os_step12.png" alt="os_step12" width="45%"> <img src="media/os_step13.png" alt="os_step13" width="45%">
<img src="media/os_step14.png" alt="os_step14" width="45%"> <img src="media/os_step15.png" alt="os_step15" width="45%">

Users and passwords.

<img src="media/os_step16.png" alt="os_step16" width="45%"> <img src="media/os_step17.png" alt="os_step17" width="45%">
<img src="media/os_step18.png" alt="os_step18" width="45%"> <img src="media/os_step19.png" alt="os_step19" width="45%">
<img src="media/os_step20.png" alt="os_step20" width="45%"> <img src="media/os_step21.png" alt="os_step21" width="45%">

Configure LVM storage.

<img src="media/os_step22.png" alt="os_step22" width="45%"> <img src="media/os_step23.png" alt="os_step23" width="45%">
<img src="media/os_step24.png" alt="os_step24" width="45%"> <img src="media/os_step25.png" alt="os_step25" width="45%">
<img src="media/os_step26.png" alt="os_step26" width="45%"> <img src="media/os_step27.png" alt="os_step27" width="45%">

Package manager.

<img src="media/os_step28.png" alt="os_step28" width="45%"> <img src="media/os_step29.png" alt="os_step29" width="45%">
<img src="media/os_step30.png" alt="os_step30" width="45%"> <img src="media/os_step31.png" alt="os_step31" width="45%">
<img src="media/os_step32.png" alt="os_step32" width="45%">

From software, unselect everything except **SSH server** and **standard system
utilities**.

<img src="media/os_step33.png" alt="os_step33" width="45%">

Bootloader.

<img src="media/os_step34.png" alt="os_step34" width="45%"> <img src="media/os_step35.png" alt="os_step35" width="45%">
<img src="media/os_step36.png" alt="os_step36" width="45%">

## OS configuration

After the install finishes and the VM reboots, log in and apply the steps below.
Sections 1 and 3 are mandatory for every VM; section 2 is only needed if static
networking was not configured during install.

### 1. General (mandatory)

Connect as the `root` user, then:

```bash
# Install the base tools we rely on
apt install vim sudo curl

# Allow the 'debian' user to use sudo
usermod -aG sudo debian
```

Log out and back in as `debian` so the new group membership takes effect.

### 2. Static networking (only if not set during install)

Edit the interfaces file:

```bash
vi /etc/network/interfaces
```

Add a static configuration for your interface (check the real name with `ip a`,
here it is `ens18`):

```conf
iface ens18 inet static
    address 192.168.20.70/24
    gateway 192.168.20.1
```

> [!WARNING]
> Set `address` and `gateway` to the correct values for the network this VM
> belongs to. The values above are only an example.

Apply it:

```bash
systemctl restart networking
```

### 3. Disable IPv6

IPv6 is always disabled on EFDI VMs. Create a sysctl drop-in file:

```bash
sudo vi /etc/sysctl.d/99-disable-ipv6.conf
```

Contents:

```conf
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
```

Reload sysctl so the change takes effect immediately:

```bash
sudo sysctl --system
```
