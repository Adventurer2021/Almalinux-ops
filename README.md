# almalinux-ops

Setup and provisioning notes for the AlmaLinux 10 box (`ryost@AlmaLinux`).

## Claude Code CLI

AlmaLinux is RHEL-based (`dnf`, not `apt`). Claude Code is a Node.js CLI installed via npm.

```bash
# Node.js 20.x via NodeSource
curl -fsSL https://rpm.nodesource.com/setup_20.x | sudo bash -
sudo dnf install -y nodejs

# Claude Code CLI
npm install -g @anthropic-ai/claude-code
claude
```

If `npm install -g` hits permission errors, point npm's global prefix at your home dir instead of using `sudo npm`:

```bash
mkdir -p ~/.npm-global
npm config set prefix ~/.npm-global
echo 'export PATH=~/.npm-global/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
npm install -g @anthropic-ai/claude-code
```

## signal-cli

`signal-cli` is a standalone Java app (not packaged for dnf) — needs a JRE plus the release tarball from GitHub.

**Java version note:** the current `signal-cli` release (0.14.8) requires **Java 25** (class file version 69). AlmaLinux 10's `java-21-openjdk-headless` (version 65) is not new enough — installing it first will produce:

```
UnsupportedClassVersionError: org/asamk/signal/Main has been compiled by a more recent version
of the Java Runtime (class file version 69.0), this version only recognizes up to 65.0
```

Install Java 25 instead (or alongside, then switch the default with `alternatives`):

```bash
sudo dnf install -y java-25-openjdk-headless

# If java-21 is already installed and set as default, switch to 25:
sudo alternatives --set java /usr/lib/jvm/java-25-openjdk/bin/java
java --version   # confirm it now reports 25.x
```

Install signal-cli itself:

```bash
SIGNAL_CLI_VERSION=$(curl -s https://api.github.com/repos/AsamK/signal-cli/releases/latest | grep -Po '"tag_name": "v\K[^"]*')

curl -L -o /tmp/signal-cli.tar.gz \
  "https://github.com/AsamK/signal-cli/releases/download/v${SIGNAL_CLI_VERSION}/signal-cli-${SIGNAL_CLI_VERSION}.tar.gz"

sudo tar xf /tmp/signal-cli.tar.gz -C /opt
sudo ln -sf /opt/signal-cli-${SIGNAL_CLI_VERSION}/bin/signal-cli /usr/local/bin/signal-cli

signal-cli --version
```

Optional, for rendering the linking QR code in-terminal:

```bash
sudo dnf install -y qrencode
```

### Linking as a secondary device

```bash
signal-cli link -n "device-name"
```

This prints an `sgnl://linkdevice?...` URI — **treat it as a one-time bearer credential, not a link to click**. Do not paste it into a browser or share it anywhere. Approve it from the primary account instead:

- **Via phone app:** pipe the URI through `qrencode -t ansiutf8` (or `-o link.png`) and scan it from Signal's *Settings → Linked Devices → Link New Device*.
- **Via another signal-cli instance already registered as primary:**
  ```bash
  signal-cli -u <primary-phone-number-E164> addDevice --uri "sgnl://linkdevice?uuid=...&pub_key=..."
  ```

Check/audit linked devices from the primary:

```bash
signal-cli -u <phone-number-E164> listDevices
signal-cli -u <phone-number-E164> removeDevice -d <device-id>   # to unlink one
```

## email-signal-forwarder — DR standby

This machine hosts a **disaster-recovery standby** copy of the Signal email forwarder that normally runs on DreamFyre (see `dreamfyre-ops`). It exists to take over if DreamFyre goes down — **it is not meant to run at the same time as DreamFyre's copy.**

- **Source:** [`Adventurer2021/signal-tools`](https://github.com/Adventurer2021/signal-tools) — same scripts as DreamFyre (`signal_email_forwarder.py`, `mailfilter.py`, `mailfilter_tui.py`).
- **Layout on this host (unlike DreamFyre, these are separate copies, not a symlink):**
  - `/home/ryost/signal_email_forwarder.py` — the deployed copy the service actually runs.
  - `/home/ryost/ops/signal_email_forwarder.py` — dev/tracked copy. **Keep these in sync manually** — editing one does not update the other.
  - `/usr/local/bin/mailfilter.py` — symlink → `/home/ryost/ops/mailfilter.py`, so it's callable bare from anywhere on `PATH`.
  - `/home/ryost/ops/mailfilter_tui.py` — Textual TUI front-end, prototype, not linked onto `PATH`. Neither `mailfilter.py` nor `mailfilter_tui.py` is scheduled anywhere (no cron, no systemd) — both are run manually.
- **Service:** `email-signal-forwarder.service` (`/etc/systemd/system/`), description `"Signal Email Forwarder (DR standby - AlmaLinux)"`.
  ```ini
  [Unit]
  Description=Signal Email Forwarder (DR standby - AlmaLinux)
  After=network-online.target docker.service
  Wants=network-online.target

  [Service]
  Type=simple
  User=ryost
  WorkingDirectory=/home/ryost
  ExecStart=/usr/bin/python3 /home/ryost/signal_email_forwarder.py
  Restart=on-failure
  RestartSec=10

  [Install]
  WantedBy=multi-user.target
  ```
  **Normal state: stopped and disabled.** It was found running for ~14h alongside DreamFyre's own active instance (2026-09-14) — both were forwarding the same mailboxes to the same Signal number simultaneously — and was stopped. Only start it during a confirmed DreamFyre outage, and stop it again once DreamFyre recovers:
  ```bash
  sudo systemctl start email-signal-forwarder.service    # begin failover
  sudo systemctl stop email-signal-forwarder.service     # end failover, once DreamFyre is back
  ```

### signal-cli-rest-api (Docker)

Defined in `~/ops/signal-cli-rest-api-compose.yml`. This is an **independent linked device** on the same Signal number (`+19194005511`) as DreamFyre — paired via its own QR code, not a re-registration of DreamFyre's session. Its container mounts `~/.local/share/signal-cli` for account/session state. Normal state matches the forwarder service: **stopped**, started only during failover.

**Important — data directory conflict:** this host also has a separate, standalone `signal-cli` binary installed (`/usr/local/bin/signal-cli → /opt/signal-cli-0.14.8/bin/signal-cli`, see the CLI section above). It shares no code with the Docker container, but both would use the *same* `~/.local/share/signal-cli` config directory by default. **Never run the host `signal-cli` binary and the `signal-cli-rest-api` container against that directory at the same time** — concurrent access risks session/lock corruption on the linked-device state. If you need the CLI for ad-hoc use (e.g. re-linking, listing devices), stop the container first.
