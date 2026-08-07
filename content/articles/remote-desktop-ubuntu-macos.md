---
publish: true
created: 2026-07-30
modified: 2026-08-07T09:59:04.689Z
---

# Remote desktop from macOS to headless Ubuntu 26.04

Ubuntu Desktop 26.04 includes GNOME Remote Desktop, so a headless Ubuntu machine can provide a graphical login without installing another desktop environment or a third-party remote desktop server.

This guide configures:

- GNOME Remote Desktop on Ubuntu
- RDP remote login before any local user session is open
- automatic startup after a reboot
- Thincast Client on macOS

The resulting connection is:

```text
Thincast on macOS → RDP → GNOME Remote Desktop → Ubuntu login
```

SSH remains the preferred interface for routine server administration. Remote desktop is useful when a graphical application or the full GNOME environment is required.

## Why RDP instead of VNC?

Ubuntu 26.04 uses GNOME on Wayland. Traditional standalone VNC servers such as TigerVNC create an X11 display and still need an X11-compatible desktop environment or window manager. A common solution is to install Xfce alongside GNOME, but that adds software and produces a different desktop from the normal Ubuntu session.

GNOME Remote Desktop is Ubuntu's built-in Wayland-aware server. Its **Remote Login** mode integrates with the GNOME Display Manager and works before anyone has logged in locally. Remote Login is available through RDP.

macOS Screen Sharing only supports VNC, so it cannot connect to this RDP service. [Thincast Client](https://thincast.com/en/products/client) is a free macOS RDP client based on FreeRDP.

## Prerequisites

This setup assumes:

- Ubuntu Desktop 26.04 LTS with GNOME
- working SSH access
- the Ubuntu machine and Mac are on the same trusted LAN
- the Ubuntu hostname is `elitedesk`

Verify that GNOME Remote Desktop is installed:

```sh
sudo grdctl --system status
```

An unconfigured installation reports that the system unit is inactive and RDP is disabled.

## 1. Prepare the service directory

The system-wide remote login service runs as the `gnome-remote-desktop` user. Confirm its home directory:

```sh
getent passwd gnome-remote-desktop
```

On Ubuntu 26.04, it is `/var/lib/gnome-remote-desktop`. Create its private data directory:

```sh
sudo install -d \
  -o gnome-remote-desktop \
  -g gnome-remote-desktop \
  -m 700 \
  /var/lib/gnome-remote-desktop/.local/share/gnome-remote-desktop
```

## 2. Generate an RDP TLS certificate

RDP uses TLS to encrypt the connection. Install the FreeRDP certificate utility:

```sh
sudo apt update
sudo apt install winpr-utils
```

Generate a key and self-signed certificate as the service user:

```sh
sudo -u gnome-remote-desktop winpr-makecert \
  -silent \
  -rdp \
  -path /var/lib/gnome-remote-desktop/.local/share/gnome-remote-desktop \
  tls
```

This creates:

```text
/var/lib/gnome-remote-desktop/.local/share/gnome-remote-desktop/tls.key
/var/lib/gnome-remote-desktop/.local/share/gnome-remote-desktop/tls.crt
```

Register both files with GNOME Remote Desktop:

```sh
sudo grdctl --system rdp set-tls-key \
  /var/lib/gnome-remote-desktop/.local/share/gnome-remote-desktop/tls.key

sudo grdctl --system rdp set-tls-cert \
  /var/lib/gnome-remote-desktop/.local/share/gnome-remote-desktop/tls.crt
```

On a machine without an available TPM interface, `grdctl` may print:

```text
Init TPM credentials failed ... using GKeyFile as fallback.
```

This is an expected fallback and does not prevent Remote Desktop from working.

## 3. Set the remote-login credentials

Configure the credentials used to reach the Ubuntu graphical login screen:

```sh
sudo grdctl --system rdp set-credentials
```

Enter a dedicated username and a strong, unique password. These are RDP gateway credentials, not the Ubuntu account credentials. Do not reuse the Ubuntu login password.

There are therefore two authentication stages:

1. Connect to GNOME Remote Desktop with the dedicated RDP credentials.
2. Log into GNOME with the normal Ubuntu username and password.

## 4. Enable remote login at boot

Enable RDP:

```sh
sudo grdctl --system rdp enable
```

Enable and start both the GNOME Display Manager and the system-wide remote desktop service:

```sh
sudo systemctl enable --now gdm.service gnome-remote-desktop.service
```

This makes Remote Login available after a reboot without anyone opening a local session.

## 5. Verify the server

Check the configuration:

```sh
sudo grdctl --system status
```

The important values are:

```text
Overall:
        Unit status: active
RDP:
        Status: enabled
        Port: 3389
        Authentication methods: credentials
```

The output also displays the TLS certificate fingerprint. Use it to verify the certificate if the client shows a certificate warning on the first connection.

Check the firewall:

```sh
sudo ufw status
```

No firewall rule is required when UFW is inactive. If a firewall is enabled later, allow TCP port `3389` only from the trusted local network or VPN. Never forward RDP port `3389` directly from the internet.

## 6. Connect from macOS with Thincast

Download and install [Thincast Client for macOS](https://thincast.com/en/products/client).

Create an RDP connection with:

- Host: `elitedesk`, `elitedesk.local`, or the machine's LAN IP address
- Port: `3389`
- Username: the dedicated RDP username
- Password: the dedicated RDP password

Accept the self-signed certificate only after comparing its fingerprint with the value from:

```sh
sudo grdctl --system status
```

After the RDP connection succeeds, the Ubuntu graphical login screen appears. Log in with the normal Ubuntu account.

Remote Login works best when that Ubuntu user is logged out locally. If the account already has an active graphical session, GNOME may offer to terminate it before opening the remote session.

## 7. Test unattended startup

Reboot the server:

```sh
sudo reboot
```

Do not log in locally. Wait for the machine to return to the network, then verify SSH:

```sh
ssh tom@elitedesk
```

Check the remote desktop service:

```sh
systemctl is-active gnome-remote-desktop.service
```

It should report:

```text
active
```

Finally, reconnect with Thincast. If both SSH and RDP work before any local login, the machine is ready for unattended headless operation.

## Security notes

- Keep SSH enabled as a recovery and administration path.
- Use separate RDP and Ubuntu passwords.
- Keep RDP restricted to the trusted LAN.
- For access away from home, connect through a VPN such as WireGuard or Tailscale instead of exposing port `3389`.
- A UPS is advisable for a server that holds important state or data.

## Next step

Configure WireGuard VPN to access the server remotely: [[wireguard-vpn-server-ubuntu]]

## References

- [Share your desktop remotely — Ubuntu Desktop 26.04 documentation](https://ubuntu.com/desktop/docs/en/26.04/how-to/share-your-desktop-remotely/)
- [GNOME Remote Desktop configuration](https://github.com/GNOME/gnome-remote-desktop/blob/main/README.md)
- [Thincast Remote Desktop Client](https://thincast.com/en/products/client)
