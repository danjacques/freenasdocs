# VPN Client Jail

This is an outline of the VPN client jail that the author is using on their NAS.
It includes:

- OpenVPN Client
- Kill Switch
- A `transmission` Client

[[TOC]]

## Overview

A BSD jail gets its own network stack, making it an ideal lightweight option to
run a VPN and software that wants to use that VPN without interfering with your
other NAS operations. By running a VPN and associated downloading and browsing
software load, one is able to browse the Internet and download files under the
protectins offered by your VPN provider.

It is important to note that VPN connections occaionaly terminate, and that when
this happens the default behavior of the OS and its software is to fall back on
the base network and route, potentially leaking identifying or private
information. This configuration includes a Kill Switch which uses the BSD
firewall to offer a hard guarante that no application data will be sent over any
connection besides the VPN.

## Instructions

Before beginning, make sure that the following is in order:

- You should be running the latest FreNAS version. Minimally, you should be
  using a version at least as new as 11.2-RC1\. See [History](#history) for more
  details.
- Download your OpenVPN configuration files, including:

  - Primary config (`openvpn.conf`)
  - Any keys and files (e.g., `ca.pem`, `tls.key`).
  - Your authentication information (e.g., username/password, private key,
    shared secret, etc.).

- Remove any hacks you put in place for previous jails (e.g., `devfs` rules on
  pre-initialization, `sysctl` rules); the are not needed and may be detrimental
  at this point.

Throughout this guide, we will be using a few variables. Please substitute your
own parameters in for these values:

- `NAME`: Your jail's name. I used `vpn`.
- `IP4`: You jail's IPv4 network address and netmask. I tested using
  `192.168.1.230/24`.
- `DEFAULTROUTE_IP4`: Your default IPv4 route, required by VNET and for
  networking. I tested using `192.168.1.1`.
- `DOWNLOADS`: The path on your NAS filesystem were downloads should be placed.
- `DOWNLOADS_UID` and `DOWNLOADS_GID`: The user and group ID that Transmission
  will use. This must be able to create files and write to `DOWNLOADS`. By
  default, the `transmission` user will get UID/GID **2000**.

In command-line snippets, these variables will be prefixed with a `$` and
enclosed in curly braces (e.g., `${NAME}`), to distinguish them from the rest of
the text.

### Create your jail

First, we will create a new `iocage` jail called `NAME`. It will have the
following notable properties:

- It will use `VNET`, as an independent virtual networking stack and routing
  table are necessary to run a VPN in a jail.
- We will set `allow_tun=1` to allow the jail to create `tun` devices.
- It will have an IPv4 address. The author hasn't attempted this with IPv6, but
  it should probably work with appropriate adaptations.
- It will mount the `DOWNLOADS` path, so that it can write to that path.
- It will automatically start on boot.

First, identify the jail environment release for your current system. The author
isn't aware a better way to do this besides running the following command and
looking at the resulting prompt. After running this command, type "EXIT" to
quit.

```sh
sudo iocage fetch
```

Look for the line:

```
Press [Enter] to fetch the default selection: (11.2-RELEASE)
```

The value in parentheses (in this case, `11.2-RELEASE`) is the current release,
hereafter `${RELEASE}`.

Create a new jail with the desired properties:

```sh
sudo iocage create \
    -n ${NAME} \
    -r ${RELEASE} \
    boot=on \
    vnet=on \
    allow_tun=1 \
    'ip4_addr=vnet0|${IP4},lo0|127.0.0.1' \
    defaultrouter=${DEFAULTROUTE_IP4}
```

> **NOTE:** The single quotes around `ip4_addr` are necessary, else your shell
> may interpret the `|` charachter as a shell pipe.

This will create your new jail.

Create a mount point in your jail for your `DOWNLOADS` mount. Note that this
will start your jail; we'll stop it afterwards since we're still building it.

```sh
sudo iocage exec ${NAME} -- mkdir /mnt/downloads
sudo iocage stop ${NAME}
```

Now, we will mount your `DOWNLOADS` directory into it.

```sh
sudo iocage fstab \
    -a \
    testvpn \
    ${DOWNLOADS} /mnt/downloads nullfs rw 0 0
```

Now that your jail is ready, enter its console. The remainder of this guide will
take place from within your jail's console.

Finally, we'll install the software that you'll need. Note that this will start
the jail back up again, this time with `DOWNLOADS` mounted.

```sh
sudo iocage pkg ${NAME} install -y  \
    openvpn \
    transmission-daemon \
    transmission-web
```

### Enter Console

For the remainder of this section, we will be running commands from within the
jail, whereas before we were entering them from the NAS. Enter the jail console
by running:

```sh
sudo iocage console ${NAME}
```

> **NOTE**: You can always return to the NAS by running the `exit` command from
> within the jail.

### OpenVPN

This section will detail how to configure OpenVPN. Every user's configuration
details will be different, so pay attention to the intent of the commands and
adapt as necessary.

We will be storing your VPN files in: `/usr/local/etc/openvpn`. Notably, your
OpenVPN configuration file must be named: `/usr/local/etc/openvpn/openvpn.conf`.

```sh
mkdir /usr/local/etc/openvpn
```

Copy all of your VPN configuration into the jail at `/usr/local/etc/openvpn/`.
See [Tips/Copying Files](#copying-files) for ideas on how to copy files from
your NAS into your jail. For VPN-provider-specific details, see the [VPN
Providers](#vpn-providers) section.

After everything is in place, edit your VPN configuration to point to the
correct paths.

Finally, enable your VPN by adding the following lines to your `/etc/rc.conf`
file:

```sh
# Enable OpenVPN.
openvpn_enable="YES"
openvpn_if="tun"
```

Now, start your VPN.

```sh
/usr/local/etc/rc.d/openvpn start
```

You can confirm that it has connected by [Checking your Jail's IP](#checking-ip)
and observing that it differs from your NAS's IP.

If your VPN is not connecting, you may be able to see why by looking at the log
information in `/var/log/messages`.

### Kill Switch

Now that the VPN is running, we will set up and test the Kill Switch.  The goal
of this configuration is:

1. Allow local network traffic as usual.
2. Permit connecting to the VPN server.
3. Block all other traffic unless it passes through the VPN's `tun0` interface.

The configuration that the author uses follows, stored in
`/usr/local/etc/openvpn/ipfw.rules`. Feel free to carefully tweak as needed:

```sh
#!/bin/sh
##
# OpenVPN Kill Switch Configuration.
#
# From:
# https://github.com/danjacques/freenasdocs
##

. /etc/network.subr

RULE_NO=1000
fwcmd="/sbin/ipfw"
add_fw() {
  ${fwcmd} add ${RULE_NO} $*
  RULE_NO=$((${RULE_NO}+1))
}

# Flush all current rules before we start.
${fwcmd} -f flush

# Enable loopback.
add_fw allow ip from any to any via lo0

# Enable VPN traffic.
add_fw allow ip from any to any via tun*

# Internal Routing
#
# Change these addresses accordingly for your internal network and netmask.
add_fw allow log ip from any to 192.168.1.0/24 keep-state

# Allow DNS traffic.
#
# OpenVPN configs may use host names, and we'll need to look these up.
# Default route.
add_fw allow log udp from any to any dst-port 53 keep-state

# Allow traffic on OpenVPN UDP port.
#
# If you're using TCP VPN and/or a different port, update accordingly. Consult
# your OpenVPN config for details.
add_fw allow log udp from any to any dst-port 1198 keep-state

# Cleanup rules.
RULE_NO=4000
add_fw allow ip from 127.0.0.1 to any

# VPN Network Access.
RULE_NO=5000
add_fw allow ip from 10.0.0.0/7 to any
add_fw allow ip from any to 10.0.0.0/7

# Block everything else.
RULE_NO=65534
add_fw deny log ip from any to any
```

> **NOTE:** This firewall configuration whitelists the default OpenVPN port.
> This is not as airtight as it could be, since any traffic destined to this
> port will be permitted. Feel free to hard-code your VPN provider's IP(s). The
> author finds that maintaining a list of those IPs and updating them is
> tedious, and that this compromise is acceptable.

After writing the config, enable your firewall by adding the following lines to
your `/etc/rc.conf` file:

```sh
# Enable Firewall.
firewall_enable="YES"
firewall_script="/usr/local/etc/openvpn/ipfw.rules"
```

Now, start your firewall.

```sh
/etc/rc.d/ipfw start
```

Since your VPN is connected from the previous step, you should be able to access
the Internet. You can test this using `curl`, for example [Checking your Jail's
IP](#checking-ip). Now, test the kill switch by disconnecting your VPN:

```sh
/usr/local/etc/rc.d/openvpn stop
```

You should no longer be able to accesss the Internet! Start your VPN to ensure
that it can still connect, and that Internet access is available again:

```sh
/usr/local/etc/rc.d/openvpn start
```

Your kill switch is good to go, and you are safe if your VPN suddenly
disconnects.

### Transmission

Now that your VPN is secure, you can enable Transmission and configure it to
download to your mounted `DOWNLOADS` directory. Add the following lines to your
`/etc/rc.conf` file:

```sh
transmission_enable="YES"
transmission_download_dir="/mnt/downloads"
```

## VPN Providers

Without getting too into the weeds here, this section will detail support for at
least one VPN provider. While the specifics here may differ between providers,
much of this information should generally translate between OpenVPN-compatible
VPN providers.

### PrivateInternetAccess

The author uses the
[PrivateInternetAccess](https://www.privateinternetaccess.com/) (PIA) VPN
provider. Consequently, these instructions are the ones that the author tested
this documentation against.

PIA stores their current configuration files here:
<https://www.privateinternetaccess.com/openvpn/openvpn.zip>

We'll need the following packages installed in order to install PIA configs:

```sh
sudo iocage pkg ${NAME} install -y curl unzip

# Enter your jail's console.
sudo iocage console ${NAME}
```

Create a file with your VPN username and password in it at
`/usr/local/etc/openvpn/auth.txt`.

```
echo "${USERNAME}" > /usr/local/etc/openvpn/auth.txt
echo "${PASSWORD}" >> /usr/local/etc/openvpn/auth.txt
```

Download and decompress the PIA OpenVPN configs:

```sh
# Download PIA configs to /root/pia
mkdir /root/pia
cd /root/pia
curl https://www.privateinternetaccess.com/openvpn/openvpn.zip | unzip -
```

The archive will contain several files, one for each PIA region that you connect
to. Choose the region that you want to use and copy it to your OpenVPN
configuration directory.

```sh
cp CA\ Montreal.ovpn /usr/local/etc/openvpn/openvpn.conf
```

Add the following to the **bottom** of your copied file,
`/usr/local/etc/openvpn/openvpn.conf`, to enable automatic login:

```
# Automatic login (PIA credentials)
auth-user-pass /usr/local/etc/openvpn/auth.txt
auth-nocache
```

Finally, set your file permissions to be acceptably restrictive for OpenVPN.

```sh
chmod 0600 -R /usr/local/etc/openvpn/
```

## Tips

### Copying Files

Copying files from your NAS to a jail involves finding the jail's root
directory. `iocage` stores the jail data at a specific path underneath the ZFS
root that it uses. You can find this path by looking for
`/mnt/<volume>/iocage/jails/${NAME}/root/`.

Alternatively, if you have already configured your `DOWNLOADS` mount, you can
see the path in `fstab`:

```sh
sudo iocage fstab -e ${NAME}
```

The second entry on a line will include the jail's root.

After identifying the root, directory run `cp` as root to copy the data in.

```sh
# Copy "file.txt" to the path "/root/file.txt" in the jail.
sudo cp /path/to/file.txt /mnt/<volume>/iocage/jails/${NAME}/root/root/
```

### Checking IP

You can check your IP using `curl` and the free `ifconfig.me` service.

First, install `curl` into your jail:

```sh
sudo iocage pkg ${NAME} install -y curl
```

Then, use `curl` to query your public IP address:

```sh
curl ifconfig.me
```

## Troubleshooting

### FreeBSD ifconfig failed

If you are getting an error message like this:

```
FreeBSD ifconfig failed: external program exited with error status: 1
```

The author encountered this when running multiple VPN jails on the same system.
One secures `tun0`, and the other sees it, thinks that it created it, and
attempts to configure it. This is likely a bug in `iocage` `devfs` rules.

To mitigate, edit your OpenVPN config for all jails and explicitly specify a
`tun` device. Open `/usr/local/etc/openvpn/openvpn.conf` and make this change:

```
## Old Line (conflicting).
# dev tun
## New Line (specific, this jail gets tun32).
dev tun32
```

Give different `tun` numbers to different jails to force them to be
non-conflicting.

## History

The following is the author's experiences using OpenVPN client on FreeNAS.

The author started using an OpenVPN client on FreeNAS 9, which used jails backed
by Warden. Setup was reatively straightforward, and didn't require and special
handling.

With FreeNAS 11, the author switched over to `iocage` which, at the time, was
fairly unfriendly to out-of-the-box VPN setup due to jail configuration issues
in particular with the `tun` logical network devices that OpenVPN uses.
Eventually, with the help of FreeNAS forums, the author was able to set up a VPN
client using a pre-initialization `devfs` hack.

After an upgrade to 11.1-RC4, the hack stopped working, and additional `devfs`
hacks were required.

In 11.2-BETA2, none of the hacks worked. As documented in [Bug
#40872](https://redmine.ixsystems.com/issues/40872), iXsystems' Brandon
Schneider worked with the community to identify and formally fix the `iocage`
`tun` device issues once and for all via the `allow_tun` configuration flag.
This flag landed in 11.2-RC1, and thereafter no hacks are necessary!

As of 11.2-RELEASE-U1, the flag is not available in the UI, and requires console
operation to set.

## Extensions

Here are some ideas that you can use to extend the utility of your VPN jail.

> **NOTE**: It is recommended that you run the minimum set of software that you
> need to inside the jail, in order to simplify maintenance and minimize the
> possibility of misconfiguration. To this end, only install services that need
> to (a) send network traffic (b) with the protection of your VPN. For example,
> the author runs their Plex server in a different jail entirely.

### Proxy

One can install a Squid proxy server in the jail in order to easily provide
client access to the VPN to any system that can access the jail. The author uses
this as a simple way to gain VPN protections without having to set up their VPN
on every system.

You can do this by installing `squid3`:

```sh
sudo iocage pkg ${NAME} install -y squid3
```

After installing, `squid` will run on port 3128 by default.

```sh
# Enable Squid Proxy.
squid_enable="YES"
```

Now, start the proxy.

```sh
/usr/local/etc/rc.d/squid start
```
