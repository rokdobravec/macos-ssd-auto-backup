

```markdown
# Auto Backup to External SSD - macOS

Automatically syncs your `Documents` folder to an external SSD whenever it's connected. Perfect for keeping a real-time backup without thinking about it.

## Features

- **Automatic** – runs when you plug in your SSD
- **Silent** – one notification per connection, not every 30 seconds
- **Safe** – overwrites existing files, never deletes anything on the SSD
- **Efficient** – uses `rsync`, only copies changed files
- **Zero interaction** – works in the background

## Requirements

- macOS (tested on Apple Silicon M1/M2/M3, works on Intel too)
- External SSD (or any USB drive) formatted as APFS, exFAT, or Mac OS Extended
- Your `Documents` folder (or any folder you choose)

## What this does

| Action | Result |
|--------|--------|
| You plug in your SSD | Script runs automatically |
| Files changed in `~/Documents` | Copy to SSD, overwriting old versions |
| New files/folders added | Copy to SSD |
| Files deleted from SSD (by accident) | Restored from your Mac on next sync |
| Files deleted from `~/Documents` | **Stay on SSD** (no automatic deletion) |
| Extra files on SSD not in `~/Documents` | **Left untouched** |

## File structure after setup

```

~/Library/Scripts/
└── sync-to-ssd.sh

~/Library/LaunchAgents/
└── com.user.ssd-sync.plist

~/Library/Scripts/
└── sync-to-ssd.log

```

## Setup Instructions

### Step 1: Create the backup script

Open Terminal and run:

```bash
mkdir -p ~/Library/Scripts
```

Create the script file (replace YOUR_USERNAME and YOUR_SSD_NAME):

```bash
cat > ~/Library/Scripts/sync-to-ssd.sh << 'EOF'
#!/bin/bash
SOURCE_FOLDER="/Users/YOUR_USERNAME/Documents"
SSD_NAME="YOUR_SSD_NAME"
DEST_SUBFOLDER="Backup"

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

Important: Replace YOUR_USERNAME with your actual macOS username (run whoami in Terminal) and YOUR_SSD_NAME with your SSD's volume name as shown in Finder.

Make the script executable:

```bash
chmod +x ~/Library/Scripts/sync-to-ssd.sh
```

Step 2: Create the automation (Launch Agent)

Create the plist file (replace YOUR_USERNAME):

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

Step 3: Grant necessary permissions

Try these options in order:

Option A (Recommended): Grant Files and Folders access

1. Open System Settings → Privacy & Security → Files and Folders
2. Click the lock to make changes
3. Click the + button
4. Press Cmd + Shift + G and type /bin/bash
5. Click Open and ensure Documents Folder and Removable Volumes are toggled ON

Option B (if Option A doesn't work): Grant Full Disk Access

1. Open System Settings → Privacy & Security → Full Disk Access
2. Click the lock → + → Cmd + Shift + G → /bin/bash
3. Click Open and toggle it ON

Step 4: Load the automation

```bash
launchctl load ~/Library/LaunchAgents/com.user.ssd-sync.plist
```

Verify it's running:

```bash
launchctl list | grep ssd-sync
```

Step 5: Test it

1. Unplug your SSD
2. Plug it back in
3. Wait 30 seconds
4. Check the log:
   ```bash
   cat ~/Library/Scripts/sync-to-ssd.log
   ```

You should see a notification banner confirming the backup.

Customization

Change the source folder:
Edit ~/Library/Scripts/sync-to-ssd.sh and change SOURCE_FOLDER.

Change the destination subfolder:
Edit DEST_SUBFOLDER variable. Use "" for root of SSD.

Change how often the script runs:
Edit ~/Library/LaunchAgents/com.user.ssd-sync.plist and change the StartInterval value (in seconds). Then reload:

```bash
launchctl unload ~/Library/LaunchAgents/com.user.ssd-sync.plist
launchctl load ~/Library/LaunchAgents/com.user.ssd-sync.plist
```

Disable notifications:
Remove or comment out the osascript block in sync-to-ssd.sh.

Troubleshooting

"Operation not permitted" error:
Try Option A first. If that doesn't work, use Option B from Step 3. Then unplug and replug your SSD.

Script runs but nothing copies:

1. Check your SSD volume name: ls /Volumes (must match exactly, case-sensitive)
2. Verify source folder exists: ls ~/Documents

No notification appears:

1. Check if script ran: cat ~/Library/Scripts/sync-to-ssd.log
2. Check System Settings → Notifications → Script Editor → allow banners

Multiple notifications on one connection:
Delete the stale flag: rm -f /tmp/ssd_notified_YOUR_SSD_NAME

"Input/output error" when loading:
Validate the plist: plutil ~/Library/LaunchAgents/com.user.ssd-sync.plist
It should output: com.user.ssd-sync.plist: OK

Uninstalling

```bash
launchctl unload ~/Library/LaunchAgents/com.user.ssd-sync.plist
rm ~/Library/LaunchAgents/com.user.ssd-sync.plist
rm ~/Library/Scripts/sync-to-ssd.sh
rm ~/Library/Scripts/sync-to-ssd.log
rm -f /tmp/ssd_notified_*
```

How it works

1. Launch Agent – macOS runs the script every 30 seconds
2. Script execution – Checks if your SSD is mounted
3. rsync – Copies files (archive mode, preserves permissions, recursive)
4. Notification – Shows a banner once per connection (flag prevents repeats)
5. Logging – Each sync is timestamped to the log file

Privacy & Security

· The script never sends data anywhere – only local copies
· All paths are local (/Users/..., /Volumes/...)
· No internet access required or used
· Try Option A first – only use Option B if needed

License

Feel free to use, modify, and share this script. No attribution required.

---

Enjoy your automatic backups!

```


```