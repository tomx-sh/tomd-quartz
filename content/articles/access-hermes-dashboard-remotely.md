---
publish: true
created: 2026-07-30
modified: 2026-08-07T09:58:44.675Z
---

# Access the Hermes dashboard from another computer

Hermes Agent includes a web dashboard for monitoring and managing the agent. By default, the dashboard is available only on the machine running Hermes.

SSH local port forwarding makes it accessible from another computer without exposing it to the network.

## Prerequisites

You need:

- Hermes installed on the remote machine
- SSH access to that machine
- the remote machine's hostname or IP address

In the commands below, replace:

- `your-user` with your SSH username
- `your-server` with the remote machine's hostname or IP address

## 1. Start the Hermes dashboard

Connect to the remote machine:

```sh
ssh your-user@your-server
```

Start the dashboard:

```sh
hermes dashboard
```

Hermes displays its local URL:

```text
HERMES_DASHBOARD_READY port=9119
Hermes Web UI → http://127.0.0.1:9119
```

Leave this terminal open so the dashboard keeps running.

If Hermes displays a different port, use that port in place of `9119` throughout the following steps.

## 2. Create the SSH tunnel

On the computer where you want to use the dashboard, open a second terminal:

```sh
ssh -N -L 9119:127.0.0.1:9119 your-user@your-server
```

Leave this terminal open as well. It carries the dashboard traffic through the encrypted SSH connection.

## 3. Open the dashboard

On the same computer where you created the tunnel, open:

```text
http://localhost:9119
```

The Hermes dashboard should now appear in the browser.

## How the command works

The forwarding command maps a port on your computer to the Hermes dashboard on the remote machine:

```text
localhost:9119 → SSH → remote machine → Hermes on 127.0.0.1:9119
```

- `-N` creates the SSH connection without opening a remote shell.
- `-L` enables local port forwarding.
- The first `9119` is the port opened on your computer.
- `127.0.0.1:9119` is the Hermes dashboard on the remote machine.

Hermes remains bound to the remote machine's loopback interface. The dashboard is transported through SSH, so no Hermes configuration or firewall rule needs to change.

## Use a different local port

If port `9119` is already occupied on your computer, use another local port:

```sh
ssh -N -L 9120:127.0.0.1:9119 your-user@your-server
```

Then open:

```text
http://localhost:9120
```

The remote port remains `9119` because that is where Hermes is listening.

## Stop the dashboard

Press `Ctrl+C` in the tunnel terminal to close the tunnel. Press `Ctrl+C` in the Hermes terminal to stop the dashboard.
