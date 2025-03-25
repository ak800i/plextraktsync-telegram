# PlexTraktSync Telegram Notifier

A Docker wrapper for [PlexTraktSync](https://github.com/Taxel/PlexTraktSync) that monitors synchronization processes and sends Telegram notifications when errors occur.

![Telegram Notification Example](https://i.imgur.com/example.png) *(Example notification screenshot placeholder)*

## Features

- 📡 **Real-time monitoring** of PlexTraktSync logs  
- 🔔 **Instant Telegram notifications** for errors and warnings  
- 📝 **Preserves original logs** while marking script-generated messages  
- 🐳 **Easy deployment** with Docker  

## Prerequisites

Before running this tool, ensure you have the following:

- [Docker](https://docs.docker.com/get-docker/) installed  
- A [Telegram bot](https://core.telegram.org/bots#3-how-do-i-create-a-bot) with a bot token  
- Your **Telegram chat ID** (find it via `@userinfobot`)  
- A working **PlexTraktSync configuration**  

## Quick Start

### 1️⃣ Create a `docker-compose.yml` file

```yaml
version: '3'
services:
  plextraktsync-watch-telegram:
    image: ghcr.io/taxel/plextraktsync
    container_name: plextraktsync-watch-telegram
    restart: on-failure:2
    volumes:
      - /path/to/your/config:/app/config
      - /path/to/your/scripts/custom_plextraktsync.sh:/app/scripts/custom_plextraktsync.sh
    environment:
      - PUID=YOUR_PUID
      - PGID=YOUR_PGID
      - TZ=Your/Timezone
      - TELEGRAM_BOT_TOKEN=your_telegram_bot_token
      - TELEGRAM_CHAT_ID=your_telegram_chat_id
    entrypoint: ["sh", "/app/scripts/custom_plextraktsync.sh"]
    command: watch
    network_mode: synobridge
```

> ⚠ **Replace** `/path/to/your/config` and `/path/to/your/scripts` with actual paths on your system.

### 2️⃣ Create the monitoring script `custom_plextraktsync.sh`

```bash
#!/bin/sh

# Function to send Telegram notification
send_telegram() {
    message="$1"
    curl -s -X POST \
        "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
        -d "chat_id=${TELEGRAM_CHAT_ID}" \
        -d "text=${message}" \
        -d "parse_mode=Markdown"
}

# Log function with identifier prefix
log_script() {
    echo "[PTS Monitor] $1"
}

# Enhanced error logging
log_error() {
    error_msg="$1"
    log_script "🚨 ERROR: $error_msg"
    send_telegram "🚨 *PlexTraktSync Error* 🚨

Error detected:
\`\`\`
$error_msg
\`\`\`"
}

# Main execution
if [ "$1" = "watch" ]; then
    log_script "Starting PlexTraktSync in watch mode..."
    
    # Monitor PlexTraktSync logs and send alerts on errors
    plextraktsync "$@" 2>&1 | while IFS= read -r line; do
        # Preserve original output
        echo "$line"
        
        # Check for errors
        if echo "$line" | grep -qiE 'error|fail|exception|warning'; then
            log_error "$line"
        fi
    done
    
    log_script "Watch mode exited with status $?"
else
    log_script "Starting PlexTraktSync..."
    if ! plextraktsync "$@"; then
        log_error "Command failed: plextraktsync $*"
    fi
    log_script "Execution completed"
fi
```

### 3️⃣ Set the script permissions

Before running the container, grant execution permissions to the script:

```bash
chmod +x /path/to/your/scripts/custom_plextraktsync.sh
```

### 4️⃣ Install required dependencies inside the container

After starting the container, run:

```bash
docker exec plextraktsync-watch-telegram apk add coreutils curl
```

---

**Now your PlexTraktSync setup is protected with real-time Telegram notifications!**
