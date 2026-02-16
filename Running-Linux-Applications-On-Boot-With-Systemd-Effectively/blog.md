I often hear: "How do I keep this running after reboot?". One of the most robust solutions on Linux is creating a ```systemd``` service. 
Since I'm a developer and maintainer of [dStats](https://pypi.org/project/dStats/), a real-time Docker monitoring tool, I will show you how to create systemd services using dStats as an example.
Also will be covering various configuration options in details.

This is your universal template for daemonizing any CLI application: Python scripts, Go binaries, Node.js tools, custom monitors, or legacy utilities. 
> Philosophy: systemd isn't magic. It's a predictable contract between your app and the OS. Master the pattern once, apply it forever.

Let’s build a robust, secure systemd service.

## The Universal 5-Step Pattern (With dStats Illustration)

### Step 1: Find Your App's Location
```bash
which dStats.server # For dStats, you MUST install it with `pip install dStats` in a virtualenv
# dStats.server: /home/youruser/venv/bin/dStats.server
```

### Step 2: Create Config File (Optional)
For dStats auth it needs `AUTH_USERNAME` and `AUTH_PASSWORD` as secrets:
> If you don't want to use auth or your app doesn't need secrets, skip this step.
```bash
sudo mkdir -p /etc/dstats # /etc/your-app
sudo nano /etc/dstats/env.conf # /etc/your-app/env.conf or config.json or config.yaml
```
Add the following lines:
```
AUTH_USERNAME=admin
AUTH_PASSWORD=yourpass
# Add all your SECRETS here
```
Secure it:
```bash
sudo chmod 600 /etc/dstats/env.conf
```

### Step 3: Create Service File
systemd files are in `/etc/systemd/system/`.
```bash
sudo nano /etc/systemd/system/dstats.service # /etc/systemd/system/your-app.service
```

Add the following content to the file:

```ini
[Unit]
Description=dStats Docker Monitor # Adjust to your app description
After=network.target # What needs to start first
Requires=docker.service # Which service needs to start before your app

[Service]
User=dstats # Adjust to your user, see Step 4 for dedicated user
Group=docker # Adjust to your group
ExecStart=/home/youruser/venv/bin/dStats.server # Adjust to your app location
Environment="USE_AUTH=true" # Adjust to your non secret env vars. e.g. `DEBUG=true SERVER_PORT=8080` etc.
EnvironmentFile=/etc/dstats/env.conf # Adjust to your secret env file location if you have any
Restart=always # on-failure, always, on-abnormal, on-watchdog
RestartSec=5

StandardOutput=journal
StandardError=journal
SyslogIdentifier=dstats # Optional, Adjust to your app name

[Install]
WantedBy=multi-user.target
```

### Step 4: Create Dedicated User - NO Shell Access (Optional)
```bash
sudo useradd -r -s /usr/sbin/nologin dstats # Adjust to your user
sudo usermod -aG docker dstats # Adjust to your user and group
```
Then update service file `User=dstats` and `Group=docker`. # Adjust to your user and group

### Step 5: Enable and Start
```bash
sudo systemctl daemon-reload # Reload systemd on any config changes
sudo systemctl enable --now dstats # Enable and start service right now
sudo systemctl status dstats # See status
sudo journalctl -u dstats -f # See logs in real time for debugging
```

## Quick Commands Reference

```bash
# Service management
systemctl start|stop|restart|status your-app

# View logs
journalctl -u your-app -f
journalctl -u your-app --since "5 min ago"

# After config changes
sudo systemctl daemon-reload
sudo systemctl restart your-app
```

## Common Options

```ini
# Resource limits
CPUQuota=50%
MemoryMax=500M

# Restart policies
Restart=on-failure
Restart=always
RestartSec=10

# Working directory
WorkingDirectory=/opt/your-app
```

That's it! Same pattern works for Python, Node.js, Go binaries, or any CLI app.

It is simpler than it looks.

Happy d[a]emonizing!
