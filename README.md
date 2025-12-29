# PodSync with One-Off Audio Feed

A custom extension to [PodSync](https://github.com/mxpv/podsync) that adds support for one-off audio file podcast feeds alongside YouTube channel conversion.

## What This Does

This project extends PodSync (which converts YouTube channels to podcast RSS feeds) with a **custom one-off audio feed generator**. Drop MP3 files into a directory and they automatically appear as a podcast feed that any podcast app can subscribe to.

### Key Features

- **Standard PodSync**: Convert YouTube channels to podcast RSS feeds
- **Custom One-Off Feed** (this repo's contribution):
  - Drop MP3 files into a directory
  - Automatic RSS feed generation
  - Auto-pruning of old files
  - Change detection (only regenerates when needed)
  - Works with any podcast app that supports local feeds

## Architecture

```
/opt/podsync/
├── docker-compose.yml    # PodSync container setup
├── config.toml           # PodSync configuration
├── scripts/              # Custom scripts (not web-accessible)
│   ├── generate-feed.sh  # RSS feed generator
│   ├── poll-audio.sh     # Change detection poller
│   └── .audio_state      # State tracking file
└── data/
    ├── <youtube-feeds>/  # Standard PodSync YouTube feeds
    └── oneoff/           # Custom one-off audio feed (web-accessible)
        ├── audio/        # Drop MP3 files here
        ├── cover.png     # Podcast artwork
        └── feed.xml      # Auto-generated RSS feed
```

## Setup

### Prerequisites

- Docker and Docker Compose
- Linux server (tested on Debian)
- YouTube API key (for YouTube channel feeds)

### Basic Installation

1. **Install PodSync via Docker:**

```bash
mkdir -p /opt/podsync/data
cd /opt/podsync

# Create docker-compose.yml
cat > docker-compose.yml <<EOF
services:
  podsync:
    container_name: podsync
    image: ghcr.io/mxpv/podsync:latest
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      - TZ=America/New_York
    volumes:
      - /opt/podsync/data:/app/data
      - /opt/podsync/config.toml:/app/config.toml
EOF

# Create basic config.toml
cat > config.toml <<EOF
[server]
port = 8080
hostname = "http://YOUR_SERVER_IP:8080"

[storage]
type = "local"

[storage.local]
data_dir = "/app/data"

[tokens]
youtube = "YOUR_YOUTUBE_API_KEY"

[database]
badger = { truncate = true, file_io = true }

[downloader]
self_update = true
timeout = 60

[log]
filename = "/app/data/podsync.log"
max_size = 40
max_age = 30
max_backups = 7
debug = false
EOF

# Start PodSync
docker compose up -d
```

2. **Add YouTube Feeds (Optional):**

Edit `/opt/podsync/config.toml` and add feeds:

```toml
[feeds.my_channel]
url = "https://www.youtube.com/channel/CHANNEL_ID"
page_size = 2
update_period = "12h"
quality = "low"
format = "audio"
clean = { days = 30 }
```

Restart: `docker compose restart`

### One-Off Audio Feed Setup

This is the **custom functionality** this repo adds to PodSync.

1. **Install dependencies:**

```bash
sudo apt update
sudo apt install -y gawk sed coreutils ffmpeg findutils
```

2. **Create directory structure:**

```bash
mkdir -p /opt/podsync/scripts
mkdir -p /opt/podsync/data/oneoff/audio
```

**Important**: These directories should be owned by your regular user account (not root), since the scripts will run as that user via cron. If you created them as root, fix ownership:

```bash
# Replace 'youruser' with your actual username
sudo chown -R youruser:youruser /opt/podsync/scripts
sudo chown -R youruser:youruser /opt/podsync/data/oneoff
```

3. **Download the scripts:**

Save `generate-feed.sh` and `poll-audio.sh` from this repo to `/opt/podsync/scripts/`

Or create them manually (see [Scripts](#scripts) section below).

4. **Make scripts executable:**

```bash
chmod +x /opt/podsync/scripts/*.sh
```

5. **Configure the feed generator:**

Edit `/opt/podsync/scripts/generate-feed.sh` and update these variables:

```bash
BASE_URL="http://YOUR_SERVER_IP:8080"
AUDIO_DIR="/opt/podsync/data/oneoff/audio"
FEED_FILE="/opt/podsync/data/oneoff/feed.xml"
MAX_EPISODES=10        # Number of episodes to show in feed
PRUNE_AFTER_DAYS=15   # Auto-delete files older than N days
```

Edit `/opt/podsync/scripts/poll-audio.sh` and update:

```bash
AUDIO_DIR="/opt/podsync/data/oneoff/audio"
STATE_FILE="/opt/podsync/scripts/.audio_state"
FEED_SCRIPT="/opt/podsync/scripts/generate-feed.sh"
```

6. **Add cover image (optional):**

```bash
# Copy a PNG image as podcast cover art
cp your-cover.png /opt/podsync/data/oneoff/cover.png
```

7. **Set up automated polling:**

```bash
crontab -e

# Add this line to check for changes every 5 minutes:
*/5 * * * * /opt/podsync/scripts/poll-audio.sh >/dev/null 2>&1
```

8. **Test the setup:**

```bash
# Manually generate feed
/opt/podsync/scripts/generate-feed.sh

# Check feed was created
cat /opt/podsync/data/oneoff/feed.xml
```

## Usage

### Adding Audio Files

Upload MP3 files to `/opt/podsync/data/oneoff/audio/`:

```bash
# Via SCP from another machine
scp audiofile.mp3 user@server:/opt/podsync/data/oneoff/audio/

# Or copy directly on server
cp /path/to/file.mp3 /opt/podsync/data/oneoff/audio/

# Or download
cd /opt/podsync/data/oneoff/audio
wget https://example.com/audiofile.mp3
```

**File naming**: The filename (minus `.mp3`) becomes the episode title, truncated to 80 characters.

### Subscribing in Podcast Apps

Add this URL to your podcast app:
```
http://YOUR_SERVER_IP:8080/oneoff/feed.xml
```

**Compatible Apps** (local network feeds):
- Apple Podcasts (iOS)
- AntennaPod (Android)
- Grover Podcast (Windows)

**Incompatible Apps** (require public URLs):
- Pocket Casts (uses server-side validation)
- Overcast (uses server-side validation)

### Accessing Feeds

- **One-off feed RSS**: `http://YOUR_SERVER_IP:8080/oneoff/feed.xml`
- **One-off audio files**: `http://YOUR_SERVER_IP:8080/oneoff/audio/`
- **YouTube feed RSS**: `http://YOUR_SERVER_IP:8080/{feed_id}.xml`
- **OPML (all feeds)**: `http://YOUR_SERVER_IP:8080/podsync.opml`
- **Logs**: `http://YOUR_SERVER_IP:8080/podsync.log`

## How It Works

### Feed Generation (`generate-feed.sh`)

1. Scans `/opt/podsync/data/oneoff/audio/` for MP3 files
2. Prunes files older than `PRUNE_AFTER_DAYS`
3. Generates RSS 2.0 feed with iTunes tags
4. Extracts audio duration using `ffprobe`
5. Sorts episodes by file modification time (newest first)
6. Limits feed to `MAX_EPISODES` most recent files
7. XML-escapes titles and adds proper metadata

### Change Detection (`poll-audio.sh`)

1. Computes SHA256 hash of audio directory listing
2. Compares against previous hash in `.audio_state`
3. Only runs `generate-feed.sh` if changes detected
4. Stores new hash for next comparison

**Efficiency**: Prevents unnecessary feed regeneration when nothing changes.

### Automated Updates (Cron)

Cron job runs every 5 minutes to check for new files and regenerate feed if needed.

## Scripts

### `generate-feed.sh`

<details>
<summary>Click to expand full script</summary>

```bash
#!/bin/sh
set -e

# =========================
# Dependency check banner
# =========================

require() {
  if ! command -v "$1" >/dev/null 2>&1; then
    echo "ERROR: Missing dependency: $1" >&2
    echo "Install with: sudo apt install -y $2" >&2
    exit 1
  fi
}

require awk gawk
require sed sed
require sha256sum coreutils
require ffprobe ffmpeg
require stat coreutils
require date coreutils
require find findutils
require rm coreutils

# =========================
# User configuration
# =========================

BASE_URL="http://192.168.1.150:1444"
AUDIO_DIR="/opt/podsync/data/oneoff/audio"
FEED_FILE="/opt/podsync/data/oneoff/feed.xml"
COVER_IMAGE="$BASE_URL/oneoff/cover.png"

MAX_EPISODES=10        # Feed-only limit
PRUNE_AFTER_DAYS=15   # Delete audio older than N days

TITLE_MAX_LEN=80

# =========================
# Helper functions
# =========================

escape_xml() {
  printf '%s' "$1" | sed 's/&/\&amp;/g; s/</\&lt;/g; s/>/\&gt;/g'
}

short_title() {
  printf '%s' "$1" | cut -c1-"$TITLE_MAX_LEN"
}

audio_duration() {
  ffprobe -v error \
    -show_entries format=duration \
    -of default=noprint_wrappers=1:nokey=1 "$1" \
    | awk '{ printf "%02d:%02d:%02d\n", int($1/3600), int(($1%3600)/60), int($1%60) }'
}

CACHE_BUSTER=$(date +%s)

# =========================
# Age-based pruning
# =========================
# Delete .mp3 files older than PRUNE_AFTER_DAYS

echo "Pruning audio files older than $PRUNE_AFTER_DAYS days (if any)..."

find "$AUDIO_DIR" \
  -type f \
  -name "*.mp3" \
  -mtime +"$PRUNE_AFTER_DAYS" \
  -print \
  -delete || true

# =========================
# Begin feed generation
# =========================

echo '<?xml version="1.0" encoding="UTF-8"?>' > "$FEED_FILE"
cat >> "$FEED_FILE" <<EOF
<rss version="2.0" xmlns:itunes="http://www.itunes.com/dtds/podcast-1.0.dtd">
<channel>
  <title>One-Off Audio Inbox</title>
  <description>Private one-off audio files</description>
  <link>$BASE_URL</link>
  <language>en-us</language>
  <itunes:author>One-Off Audio Inbox</itunes:author>
  <itunes:summary>Private one-off audio files</itunes:summary>
  <itunes:explicit>false</itunes:explicit>
  <itunes:category text="Technology"/>
  <itunes:image href="$COVER_IMAGE?v=$CACHE_BUSTER"/>
EOF

ls -t "$AUDIO_DIR"/*.mp3 2>/dev/null | head -n "$MAX_EPISODES" | while read -r FILE; do
  BASENAME=$(basename "$FILE")
  TITLE_RAW=$(short_title "${BASENAME%.mp3}")
  TITLE_ESC=$(escape_xml "$TITLE_RAW")
  SIZE=$(stat -c%s "$FILE")
  PUBDATE=$(date -R -r "$FILE")
  DURATION=$(audio_duration "$FILE")

  cat >> "$FEED_FILE" <<EOF
  <item>
    <title>$TITLE_ESC</title>
    <pubDate>$PUBDATE</pubDate>
    <guid>$BASENAME</guid>
    <itunes:duration>$DURATION</itunes:duration>
    <enclosure url="$BASE_URL/oneoff/audio/$BASENAME" length="$SIZE" type="audio/mpeg"/>
  </item>
EOF
done

echo '</channel></rss>' >> "$FEED_FILE"
```

</details>

### `poll-audio.sh`

<details>
<summary>Click to expand full script</summary>

```bash
#!/bin/sh
set -e

# =========================
# Dependency check banner
# =========================

require() {
  if ! command -v "$1" >/dev/null 2>&1; then
    echo "ERROR: Missing dependency: $1" >&2
    echo "Install with: sudo apt install -y $2" >&2
    exit 1
  fi
}

require awk gawk
require sha256sum coreutils
require ls coreutils

# =========================
# Configuration
# =========================

AUDIO_DIR="/opt/podsync/data/oneoff/audio"
STATE_FILE="/opt/podsync/scripts/.audio_state"
FEED_SCRIPT="/opt/podsync/scripts/generate-feed.sh"

# =========================
# Change detection
# =========================

NEW_STATE=$(ls -l --time-style=+%s "$AUDIO_DIR" 2>/dev/null | awk '{print $6,$7,$8,$9}' | sha256sum)

OLD_STATE=""
[ -f "$STATE_FILE" ] && OLD_STATE=$(cat "$STATE_FILE")

if [ "$NEW_STATE" != "$OLD_STATE" ]; then
  echo "$NEW_STATE" > "$STATE_FILE"
  "$FEED_SCRIPT"
fi
```

</details>

## Configuration Options

### Feed Settings

Edit `/opt/podsync/scripts/generate-feed.sh`:

```bash
MAX_EPISODES=10        # Number of episodes in feed (older files still kept)
PRUNE_AFTER_DAYS=15   # Auto-delete files older than N days
TITLE_MAX_LEN=80      # Maximum episode title length
```

### Feed Metadata

Edit the XML section in `generate-feed.sh`:

```xml
<title>One-Off Audio Inbox</title>
<description>Private one-off audio files</description>
<itunes:author>One-Off Audio Inbox</itunes:author>
<itunes:category text="Technology"/>
```

### Poll Frequency

Edit crontab to adjust polling interval:

```bash
crontab -e

# Every 1 minute (more responsive)
* * * * * /opt/podsync/scripts/poll-audio.sh >/dev/null 2>&1

# Every 5 minutes (balanced - recommended)
*/5 * * * * /opt/podsync/scripts/poll-audio.sh >/dev/null 2>&1

# Every 15 minutes (less frequent)
*/15 * * * * /opt/podsync/scripts/poll-audio.sh >/dev/null 2>&1
```

## Troubleshooting

### Feed Not Updating

```bash
# Wait up to 5 minutes for cron, or manually regenerate
/opt/podsync/scripts/generate-feed.sh

# Force change detection
rm /opt/podsync/scripts/.audio_state
/opt/podsync/scripts/poll-audio.sh
```

### Check Feed Status

```bash
# View current feed
cat /opt/podsync/data/oneoff/feed.xml

# Count episodes
grep -c "<item>" /opt/podsync/data/oneoff/feed.xml

# List audio files
ls -lh /opt/podsync/data/oneoff/audio/
```

### Missing Dependencies

```bash
sudo apt update
sudo apt install -y gawk sed coreutils ffmpeg findutils
```

### Verify Cron Job

```bash
# View crontab
crontab -l

# Enable logging for debugging
crontab -e
# Change to:
*/5 * * * * /opt/podsync/scripts/poll-audio.sh >> /tmp/poll-audio.log 2>&1

# Monitor logs
tail -f /tmp/poll-audio.log
```

### Docker Commands

```bash
# View PodSync logs
docker logs -f podsync

# Restart PodSync
docker compose restart

# Update to latest version
docker compose pull && docker compose up -d
```

## Use Cases

- **Audiobooks**: Upload chapters or full books as podcast episodes
- **Personal recordings**: Voice memos, lectures, meeting recordings
- **Downloaded podcasts**: Episodes from services without RSS feeds
- **Temporary audio**: Files you want available for a limited time
- **Private content**: Audio not suitable for public platforms
- **Podcast backlogs**: Old episodes no longer in original feeds
- **Audio newsletters**: Personal audio content distribution

## Security Notes

- **Scripts location**: Scripts are stored in `/opt/podsync/scripts/` which is **not web-accessible**. Only the generated feed and audio files in `/opt/podsync/data/oneoff/` are served via HTTP.
- **Local network**: This setup is designed for local network use. For public access, add authentication and HTTPS.
- **File permissions**: Ensure proper ownership of script and data directories.

## Contributing

This is a personal project, but contributions are welcome:

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Submit a pull request

## License

This project extends [PodSync](https://github.com/mxpv/podsync) which is licensed under MIT.

The custom one-off feed scripts in this repository are also released under the MIT License.

## Credits

- **PodSync**: [mxpv/podsync](https://github.com/mxpv/podsync) - The amazing tool that converts YouTube to podcasts
- **Custom Scripts**: One-off audio feed functionality

## Support

For PodSync-specific issues, see the [official PodSync repository](https://github.com/mxpv/podsync).

For issues with the one-off audio feed scripts, please open an issue in this repository.
