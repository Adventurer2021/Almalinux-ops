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
