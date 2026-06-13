```markdown
# 💾 Auto Backup to External SSD

<div align="center">

**Plug in. Backup. Done.**

[![macOS](https://img.shields.io/badge/macOS-13%2B-blue.svg)](https://www.apple.com/macos)
[![Apple Silicon](https://img.shields.io/badge/Apple%20Silicon-Ready-green.svg)](https://www.apple.com/m1)
[![Intel](https://img.shields.io/badge/Intel-Supported-green.svg)](https://www.apple.com)
[![License](https://img.shields.io/badge/License-MIT-lightgrey.svg)](LICENSE)

*A zero-interaction backup solution for your Mac*

</div>

---

## 📖 Overview

Ever forget to back up your Documents folder? This automation runs **every time you plug in your external SSD** – silently copying your files in the background. No buttons. No schedules. No thinking required.

It uses `rsync` to copy only what changed, shows **one notification** when connected, and never deletes anything from your SSD.

---

## ✨ Features

| Feature | Description |
|---------|-------------|
| 🔌 **Plug & Play** | Runs automatically when you connect your SSD |
| 🔕 **Silent** | One notification per connection – no spam |
| 🛡️ **Safe** | Overwrites existing files, never deletes from SSD |
| ⚡ **Efficient** | `rsync` copies only changed files |
| 📝 **Logged** | Every sync is timestamped to a log file |
| 🎛️ **Customizable** | Change source folder, sync interval, destination |

---

## 🗺️ How It Works

```

┌─────────────┐     ┌──────────────┐     ┌─────────────────┐
│  Plug in    │ ──▶ │  Launchd     │ ──▶ │  rsync copies   │
│  Your SSD   │     │  triggers    │     │  ~/Documents    │
│             │     │  every 30s   │     │  → /Volumes/SSD │
└─────────────┘     └──────────────┘     └─────────────────┘

```

| Action | Result |
|--------|--------|
| You plug in your SSD | Script runs automatically |
| Files changed in `~/Documents` | Copied to SSD, overwriting old versions |
| New files/folders added | Copied to SSD |
| Files deleted from SSD (by accident) | Restored on next sync |
| Files deleted from `~/Documents` | **Stay on SSD** (no deletion) |
| Extra files on SSD (not in source) | **Left untouched** |

---

## 📁 File Structure

```

~/Library/Scripts/
└── sync-to-ssd.sh              # The backup script

~/Library/LaunchAgents/
└── com.user.ssd-sync.plist     # The automation trigger

~/Library/Scripts/
└── sync-to-ssd.log              # Sync history log

```

---

## 🚀 Quick Setup

### 1. Create the script

```bash
mkdir -p ~/Library/Scripts
```

```bash
cat > ~/Library/Scripts/sync-to-ssd.sh << 'EOF'
#!/bin/bash
# ============================================
# Auto Backup to External SSD
# ============================================
# ⚙️ EDIT THESE THREE VARIABLES:
SOURCE_FOLDER="/Users/YOUR_USERNAME/Documents"
SSD_NAME="YOUR_SSD_NAME"           # As shown in Finder
DEST_SUBFOLDER="Backup"             # Folder on SSD (use "" for root)

# ============================================
# Don't edit below this line
# ============================================
SSD_PATH="/Volumes/$SSD_NAME"

if [ ! -d "$SSD_PATH" ]; then
    rm -f "/tmp/ssd_notified_$SSD_NAME"
fi

if [ -d "$SSD_PATH" ] && [ -d "$SOURCE_FOLDER" ]; then
    DEST_PATH="$SSD_PATH/$DEST_SUBFOLDER"
    /bin/mkdir -p "$DEST_PATH"
    /usr/bin/rsync -a --ignore-errors "$SOURCE_FOLDER/" "$DEST_PATH/"
    
    NOTIF_FLAG="/tmp/ssd_notified_$SSD_NAME"
    if [ ! -f "$NOTIF_FLAG" ]; then
        CURRENT_USER=$(/usr/bin/stat -f "%Su" /dev/console)
        if [ -n "$CURRENT_USER" ]; then
            /usr/bin/sudo -u "$CURRENT_USER" /usr/bin/osascript -e "display notification \"Your files have been backed up\" with title \"Backup Complete\" subtitle \"$SSD_NAME\" sound name \"Glass\""
        fi
        touch "$NOTIF_FLAG"
    fi
    
    echo "$(/bin/date): Synced $SOURCE_FOLDER to $DEST_PATH" >> ~/Library/Scripts/sync-to-ssd.log
fi
EOF
```

Replace YOUR_USERNAME and YOUR_SSD_NAME with your actual values.

```bash
chmod +x ~/Library/Scripts/sync-to-ssd.sh
```

2. Create the automation

```bash
cat > ~/Library/LaunchAgents/com.user.ssd-sync.plist << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.user.ssd-sync</string>
    <key>ProgramArguments</key>
    <array>
        <string>/bin/bash</string>
        <string>/Users/YOUR_USERNAME/Library/Scripts/sync-to-ssd.sh</string>
    </array>
    <key>StartInterval</key>
    <integer>30</integer>
    <key>RunAtLoad</key>
    <false/>
</dict>
</plist>
EOF
```

3. Grant permissions

macOS may block background access. Try Option A first; if you get errors, use Option B.

<details>
<summary><b>Option A: Files & Folders (Recommended)</b></summary>1. System Settings → Privacy & Security → Files and Folders
2. Click the lock to make changes
3. Click + → press Cmd + Shift + G → type /bin/bash
4. Toggle ON: Documents Folder and Removable Volumes

</details><details>
<summary><b>Option B: Full Disk Access (Fallback)</b></summary>1. System Settings → Privacy & Security → Full Disk Access
2. Click the lock → + → Cmd + Shift + G → /bin/bash
3. Toggle ON

</details>4. Load and test

```bash
launchctl load ~/Library/LaunchAgents/com.user.ssd-sync.plist
```

```bash
# Verify it's running
launchctl list | grep ssd-sync
```

Test: Unplug your SSD, plug it back in, wait 30 seconds, then check the log:

```bash
cat ~/Library/Scripts/sync-to-ssd.log
```

You should see a notification and a log entry.

---

🎛️ Customization

<details>
<summary><b>Change source folder</b></summary>Edit ~/Library/Scripts/sync-to-ssd.sh and change:

```bash
SOURCE_FOLDER="/Users/YOUR_USERNAME/Documents"
```

</details><details>
<summary><b>Change destination subfolder</b></summary>Edit DEST_SUBFOLDER variable:

· "" → copies directly to SSD root
· "MyBackups" → copies into /Volumes/SSD/MyBackups

</details><details>
<summary><b>Change sync frequency</b></summary>Edit ~/Library/LaunchAgents/com.user.ssd-sync.plist:

```xml
<key>StartInterval</key>
<integer>30</integer>   <!-- Change 30 to desired seconds -->
```

Then reload:

```bash
launchctl unload ~/Library/LaunchAgents/com.user.ssd-sync.plist
launchctl load ~/Library/LaunchAgents/com.user.ssd-sync.plist
```

</details><details>
<summary><b>Disable notifications</b></summary>Remove or comment out the osascript block in sync-to-ssd.sh.

</details>---

🔧 Troubleshooting

<details>
<summary><b>"Operation not permitted" error</b></summary>Use Option B (Full Disk Access) from Step 3, then unplug/replug your SSD.

</details><details>
<summary><b>Script runs but nothing copies</b></summary>1. Check your SSD name: ls /Volumes (must match SSD_NAME exactly, case‑sensitive)
2. Check source exists: ls ~/Documents

</details><details>
<summary><b>No notification appears</b></summary>1. Check if script ran: cat ~/Library/Scripts/sync-to-ssd.log
2. Check System Settings → Notifications → Script Editor → allow banners

</details><details>
<summary><b>"Input/output error" when loading</b></summary>The plist may be malformed. Validate:

```bash
plutil ~/Library/LaunchAgents/com.user.ssd-sync.plist
```

Should output: com.user.ssd-sync.plist: OK

</details>---

🗑️ Uninstalling

```bash
launchctl unload ~/Library/LaunchAgents/com.user.ssd-sync.plist
rm ~/Library/LaunchAgents/com.user.ssd-sync.plist
rm ~/Library/Scripts/sync-to-ssd.sh
rm ~/Library/Scripts/sync-to-ssd.log
rm -f /tmp/ssd_notified_*
```

Remove /bin/bash from Files & Folders or Full Disk Access if desired.

---

📖 Technical Deep Dive

<details>
<summary><b>How it works</b></summary>1. Launch Agent – macOS runs com.user.ssd-sync.plist every 30 seconds
2. Script execution – Checks if SSD is mounted at /Volumes/YOUR_SSD_NAME
3. rsync – Copies files from source to destination:
   · -a (archive) = preserves permissions, timestamps, recursive copy
   · --ignore-errors = continues even if some files fail
4. Notification – osascript shows a banner once per connection (flag file prevents repeats)
5. Logging – Each sync is timestamped to ~/Library/Scripts/sync-to-ssd.log

</details><details>
<summary><b>Privacy & Security</b></summary>· ✅ Copies only from ~/Documents to your SSD – nothing else
· ✅ No internet access – all local
· ✅ No data sent anywhere
· ✅ Minimal permissions needed (Files & Folders is enough; Full Disk is fallback)

</details>---

📝 License

MIT – use it, modify it, share it. No attribution required.

---