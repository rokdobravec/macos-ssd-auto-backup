```
# Auto Backup to External SSD - macOS

Automatically syncs your `Documents` folder to an external SSD whenever it's connected. Perfect for keeping a real-time backup without thinking about it.

## ✨ Features

- **Automatic** – runs when you plug in your SSD
- **Silent** – one notification per connection, not every 30 seconds
- **Safe** – overwrites existing files, never deletes anything on the SSD
- **Efficient** – uses `rsync`, only copies changed files
- **Zero interaction** – works in the background

## 📋 Requirements

- macOS (tested on Apple Silicon M1/M2/M3, works on Intel too)
- External SSD (or any USB drive) formatted as APFS, exFAT, or Mac OS Extended
- Your `Documents` folder (or any folder you choose)

## 🗂️ What this does

| Action | Result |
|--------|--------|
| You plug in your SSD | Script runs automatically |
| Files changed in `~/Documents` | Copy to SSD, overwriting old versions |
| New files/folders added | Copy to SSD |
| Files deleted from SSD (by accident) | Restored from your Mac on next sync |
| Files deleted from `~/Documents` | **Stay on SSD** (no automatic deletion) |
| Extra files on SSD not in `~/Documents` | **Left untouched** |

## 📁 File structure after setup

```

~/Library/Scripts/
└── sync-to-ssd.sh              # The backup script

~/Library/LaunchAgents/
└── com.user.ssd-sync.plist     # The automation trigger

~/Library/Scripts/
└── sync-to-ssd.log              # Backup log (auto-created)

```

## ⚙️ Setup Instructions

### Step 1: Create the backup script

Open Terminal and run:

```bash
mkdir -p ~/Library/Scripts
```

Create the script file (replace YOUR_USERNAME and YOUR_SSD_NAME):

```bash
cat > ~/Library/Scripts/sync-to-ssd.sh << 'EOF'
#!/bin/bash
# ============================================
# Auto Backup to External SSD
# ============================================
# Edit these three variables to match your setup:
SOURCE_FOLDER="/Users/YOUR_USERNAME/Documents"
SSD_NAME="YOUR_SSD_NAME"           # Exactly as shown in Finder
DEST_SUBFOLDER="Backup"             # Folder name on SSD (can be "" for root)

# ============================================
# Don't edit below this line
# ============================================
SSD_PATH="/Volumes/$SSD_NAME"

# Clean up stale notification flag if SSD not mounted
if [ ! -d "$SSD_PATH" ]; then
    rm -f "/tmp/ssd_notified_$SSD_NAME"
fi

if [ -d "$SSD_PATH" ] && [ -d "$SOURCE_FOLDER" ]; then
    DEST_PATH="$SSD_PATH/$DEST_SUBFOLDER"
    /bin/mkdir -p "$DEST_PATH"
    
    # Copy files (archive mode, ignore permission errors)
    /usr/bin/rsync -a --ignore-errors "$SOURCE_FOLDER/" "$DEST_PATH/"
    
    # Notify only once per SSD connection
    NOTIF_FLAG="/tmp/ssd_notified_$SSD_NAME"
    if [ ! -f "$NOTIF_FLAG" ]; then
        CURRENT_USER=$(/usr/bin/stat -f "%Su" /dev/console)
        if [ -n "$CURRENT_USER" ]; then
            /usr/bin/sudo -u "$CURRENT_USER" /usr/bin/osascript -e "display notification \"Your files have been backed up\" with title \"Backup Complete\" subtitle \"$SSD_NAME\" sound name \"Glass\""
        fi
        touch "$NOTIF_FLAG"
    fi
    
    # Log the sync
    echo "$(/bin/date): Synced $SOURCE_FOLDER to $DEST_PATH" >> ~/Library/Scripts/sync-to-ssd.log
fi
EOF
```

Important: Replace:

· YOUR_USERNAME with your actual macOS username (run whoami in Terminal)
· YOUR_SSD_NAME with your SSD's volume name as shown in Finder

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

macOS may block the script from accessing your Documents folder when run in the background. Try these options in order:

Option A (Recommended – least permissions): Grant Files and Folders access

1. Open System Settings → Privacy & Security → Files and Folders
2. Click the lock icon to make changes
3. Click the + button
4. Press Cmd + Shift + G and type /bin/bash
5. Click Open
6. Ensure Documents Folder and Removable Volumes are toggled ON

Option B (If Option A doesn't work): Grant Full Disk Access

If you still see Operation not permitted errors after testing, grant Full Disk Access:

1. Open System Settings → Privacy & Security → Full Disk Access
2. Click the lock icon to make changes
3. Click the + button
4. Press Cmd + Shift + G and type /bin/bash
5. Click Open and toggle it ON

Why this is safe: Your script only reads ~/Documents and writes to your external SSD. It doesn't access emails, messages, or system files.

Step 4: Load the automation

```bash
launchctl load ~/Library/LaunchAgents/com.user.ssd-sync.plist
```

Verify it's running:

```bash
launchctl list | grep ssd-sync
```

You should see output like: 12345 0 com.user.ssd-sync

Step 5: Test it

1. Unplug your SSD
2. Plug it back in
3. Wait 30 seconds
4. Check the log:
   ```bash
   cat ~/Library/Scripts/sync-to-ssd.log
   ```
5. You should see a notification banner confirming the backup

🎛️ Customization

Change the source folder

Edit ~/Library/Scripts/sync-to-ssd.sh and change:

```bash
SOURCE_FOLDER="/Users/YOUR_USERNAME/Documents"
```

to any folder you want (e.g., "/Users/YOUR_USERNAME/Desktop" or "/Users/YOUR_USERNAME/Pictures")

Change the destination subfolder

Edit the DEST_SUBFOLDER variable:

· Use "" (empty quotes) to copy directly to the root of the SSD
· Use "MyBackups" to put everything inside a folder called MyBackups

Change how often the script runs

Edit ~/Library/LaunchAgents/com.user.ssd-sync.plist and change the number:

```xml
<key>StartInterval</key>
<integer>30</integer>   <!-- Change 30 to any number of seconds -->
```

Then reload:

```bash
launchctl unload ~/Library/LaunchAgents/com.user.ssd-sync.plist
launchctl load ~/Library/LaunchAgents/com.user.ssd-sync.plist
```

Disable notifications

Remove or comment out this block from sync-to-ssd.sh:

```bash
    if [ ! -f "$NOTIF_FLAG" ]; then
        CURRENT_USER=$(/usr/bin/stat -f "%Su" /dev/console)
        if [ -n "$CURRENT_USER" ]; then
            /usr/bin/sudo -u "$CURRENT_USER" /usr/bin/osascript -e "display notification..."
        fi
        touch "$NOTIF_FLAG"
    fi
```

🔧 Troubleshooting

"Operation not permitted" error

Solution: Try Option A (Files and Folders) first. If that doesn't work, use Option B (Full Disk Access) from Step 3. Then unplug and replug your SSD.

Script runs but nothing copies

1. Check your SSD volume name – it must match exactly (case-sensitive):
   ```bash
   ls /Volumes
   ```
2. Verify the source folder exists:
   ```bash
   ls ~/Documents
   ```

No notification appears

1. Check if the script ran at all:
   ```bash
   cat ~/Library/Scripts/sync-to-ssd.log
   ```
2. If the log shows the sync happened but no notification, check System Settings → Notifications → Script Editor (or Terminal) and ensure banners are allowed.

Multiple notifications on one connection

This is already prevented by the flag system. If it still happens, delete the stale flag:

```bash
rm -f /tmp/ssd_notified_YOUR_SSD_NAME
```

"Input/output error" when loading

The plist file might be malformed. Validate it:

```bash
plutil ~/Library/LaunchAgents/com.user.ssd-sync.plist
```

It should output: com.user.ssd-sync.plist: OK

If not, recreate the plist from Step 2.

🛑 Uninstalling

To completely remove the automation:

```bash
# Stop and remove the launch agent
launchctl unload ~/Library/LaunchAgents/com.user.ssd-sync.plist
rm ~/Library/LaunchAgents/com.user.ssd-sync.plist

# Remove the script
rm ~/Library/Scripts/sync-to-ssd.sh

# Remove the log (optional)
rm ~/Library/Scripts/sync-to-ssd.log

# Remove the notification flag
rm -f /tmp/ssd_notified_*
```

You can also remove /bin/bash from Files and Folders or Full Disk Access in System Settings if desired.

📖 How it works (technical overview)

1. Launch Agent – macOS runs com.user.ssd-sync.plist every 30 seconds
2. Script execution – The script checks if your SSD is mounted at /Volumes/YOUR_SSD_NAME
3. rsync – If mounted, rsync -a copies all files from source to destination:
   · -a (archive mode) preserves permissions, timestamps, and copies recursively
   · --ignore-errors continues even if some files fail
4. Notification – osascript shows a banner once per connection (flag file prevents repeats)
5. Logging – Each sync is timestamped in ~/Library/Scripts/sync-to-ssd.log

🔒 Privacy & Security

· The script never sends data anywhere – it only copies locally to your external drive
· All paths are local (/Users/..., /Volumes/...)
· No internet access required or used
· The script only needs access to ~/Documents and your external drive
· Try Option A (Files and Folders) first – only use Option B (Full Disk Access) if needed

📝 License

Feel free to use, modify, and share this script. No attribution required.

🤝 Contributing

Found a bug or have an improvement? Open an issue or pull 

