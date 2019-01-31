# SSH Server and Tunnels as "VPN"

This is a guide on using FreeNAS's SSH service and SSH's tunneling capability to
enable secure, authenticated remote connections to your server and, through it,
your jails and internal network.

## Why?

Exposing services to the Internet is dangerous. The Internet is accessible by
anyone in the world, anywhere, and opening your network doors to the vast number
of players out there can be extremely dangerous, resulting in compromise of your
network, services, systems, and data.

When exposing a service to the Internet, one must be concerned about three
things in particular:
* **Authentication**: Make sure only people who are allowed to access your
    service can actually access it.
* **Confidentiality**: Any data that you send to your services can only be read
    by your services, and vice versa.
* **Integrity**: Any communications that you have with your services are not
    tampered with.

Doing this with the entire world as your potential adversary is far more
difficult than it seems to be. Some services offer some or all of these
features. However, a single bug in the implementation of any of these can
compromise the entire thing. You want your security features to be written,
audited, and tested by world-class experts. Most simple services (e.g.,
Transmission) simply are not. A single bug in any one of them could allow
unfettered access to your internal network!

Rather than directly expose your services directly to the Internet, you can
make your services available only to your internal network, and then install
a strong gate for authenticated remote users to join your local network to
access those services. This is what a VPN does - it allows a properly-authorized
remote system to join your local network over an encrypted channel. Now, your
services don't have to worry about external threats, since the only users that
can access them are either internal to your network or authenticated through the
VPN gateway.

OpenVPN is a heavily-audited piece of software that is up to the task. However,
setting up an OpenVPN server and provisioning clients to connect to it is pretty
in-depth and time-consuming.

Another option, discussed here, is what I call "VPN Lite". It uses SSH (which
comes with FreeNAS) as the strong gate guarding your network, and SSH's awesome
tunnelling features to allow access to your services. SSH is a lot easier to set
up, both the server (via FreeNAS) and clients (varied), than OpenVPN is.
However, it is very secure and works as a great locked door to your internal
network.

> As an aside, I used this approach for 6+ years for 8+ exposed services. After
> a bit of practice, it is pretty painless and **very** comforting to know that
> your personal network is protected by state-of-the-art software and
> encryption.

## Setup

Before we start, you'll need a few things:

* The IP addresses and ports of the services that you are exposing.
  * e.g., for Transmission, this would be the IP address of your Transmission
    jail and your Transmission port (9091 by default).
* SSH client software installed on your client device(s).
* An SSH keypair for each client device (see below).

### Create Client Keys

You'll need to create a key to identify your client (the device connecting to your network). Each SSH key comes in two parts: a private key, which authenticates your device, and a public key, which you will install on your NAS to permit the device to connect.

You will need SSH software on your client, both to connect and to generate keys. Here are some ideas:

* Windows: use [PuTTY](https://www.putty.org/) as SSH client, and [PuTTYgen](https://www.ssh.com/ssh/putty/windows/puttygen) to generate keys.
   * Note, when generating keys, there will be a large "Key" textbox. That will be the public key that you will provide to the NAS.
* Mac, Linux: use built-in `ssh` client, and [ssh-keygen](https://www.ssh.com/ssh/keygen/) to generate keys.
   * Note, generated keys will, by default, be placed in `~/.ssh/id_something` (private key) and `~/.ssh/id_something.pub` (public key). The public key file's contents will be pasted onto the NAS.
* Android: I like [JuiceSSH](https://juicessh.com/), which is both a client and can generate keys.
   * Note, you can get your public key by navigating to the generated key, long-holding, and then selecting "copy public key", which copies it to your clipboard.
* iOS: I don't have an iPhone, but there are SSH clients. [Termius](https://itunes.apple.com/us/app/termius-ssh-client/id549039908?mt=8) seems pretty well-rated.

In general, it is 100% safe to give the **public** key data to anyone. However, guard the **private** key data very closely, as anyone who has that file can access your NAS.

### FreeNAS Setup

Enable the SSH service on your FreeNAS system. Instructions for that can be found [here](https://www.ixsystems.com/documentation/freenas/11.2/services.html#ssh).

I like to make sure "Allow Password Authentication" option is **unchecked**. This requires a client to have a valid key to connect; if you allow password authentication, anyone with your password can connect. You can spend years guessing a password from anywhere in the world, but it is (effectively) impossible for someone to guess your key. Keys are significantly more secure.

Furthermore, if you allow passwords, you will need to make sure that any active account on your NAS has a good strong password. No point in doing that when you can just use super-secure keys instead.

And as a bonus, if you are using key authentication instead of password authentication, you don't have to type any password to connect ... just have the key! One-click, baby.

### Install your public Key

Pick (or [create](https://www.ixsystems.com/documentation/freenas/11.2/accounts.html#users)) a FreeNAS user to use for the remote connection. I create one myself, called `remote`.

Edit the account and scroll to the "SSH Public Key" section. Here, paste the **public** (not private!) key that you generated earlier and save. Now, your NAS will accept your private key as proof of identify for that user!

Note: if you have multiple public keys (e.g., multiple client devices that you are provisioning), you can paste multiple public keys into the "SSH Public Key" dialog. Just make sure there's an empty line in between each key and all of them will work.

### SSH Test

Try and connect to your NAS from your client device. To do this, open up your SSH client. You are connecting to `user@address`. If, for example, your user is `remote` and your NAS's local address is `192.68.1.200`, you would connect to `remote@192.168.1.200`. Generally, the generated SSH key will be used by default, but some clients will make you manually specify the SSH private key file path.

You should connect instantly and get a FreeNAS shell. If you don't, LMK what OS and SSH client you are using and I'll see if I can help.

### Tunneling

Once you can connect, the rest is easy. Tunneling is where you ask SSH to open a port on your local client system and tunnel it to a port on your SSH server (your FreeNAS box). Traffic to the client port will be sent through the SSH connection (encrypted!) and come out the other side, on your NAS. Services on the other side of the tunnel (e.g., Transmission) will think that the traffic is coming from inside your network, which is how you get around the OpenVPN routing junk.

To tunnel, you tell your SSH client, "open a local port, and have it pass through the SSH tunnel to a port on the other side." Therefore, you will need:

* The **local IP address** of that you're trying to talk to. In this case, we're trying to talk to your Transmission jail, so whatever address that is. I'll use `192.168.1.50` in this example.
* The **port** of the service that you're trying to talk to. In this case, we're doing Transmission, which is (by default) `9091`.
* A local port. This can be anything, but for sanity, I like to keep the numbers similar. We'll use `19091`, but you can use any port number, including `9091`.

Tunnel configuration depends on the client. For Linux/Mac, this would be represented as a command-line flag `-L19091:192.168.1.50:9091`. In Windows with PuTTY, you would add Transmission as a forwarded port (general instructions [here](https://stackoverflow.com/questions/4974131/how-to-create-ssh-tunnel-using-putty-in-windows)). You should be able to Google how to do this for whatever client you are using.

### Full Test

After this is done, you can test this by connecting SSH (with the tunnel!) and visiting your local port: http://localhost:19091

This should open the Transmission UI. If it does, you're set!

### Router Port Forwards

Finally, once you've tested things out locally, enable port forwarding to your SSH service, port 22. I like to use an alternate public port (e.g., 20022) just to make it slightly harder to be port scanned, although with a good security setup, it probably doesn't matter.

I would also remove any other port forwards to your NAS (or, ideally, your internal network).  Now that you have this super-secure encrypted setup, you can just add them as additional tunnels in your SSH configuration. Ideally, you will just use that one SSH port forward as your guarded gate, and do everything through SSH tunnels.
