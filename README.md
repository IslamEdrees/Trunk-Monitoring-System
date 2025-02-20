# Trunk Monitoring System

## Overview
This project provides a **trunk monitoring system** for **Asterisk** using **Bash scripting** and **Telegram notifications**. It periodically checks the SIP trunk status and alerts the user via a Telegram bot if any trunks are **UNREACHABLE** or **UNKNOWN**.

## Features
- Monitors SIP trunks using `asterisk -rx 'sip show peers'`.
- Logs the status of the trunks in `/var/log/trunks_status.log`.
- Sends real-time notifications to **Telegram** when a trunk goes down.
- Uses **SSH key authentication** for secure connection to the Asterisk server.
- Configurable to run every minute or at custom intervals.

---

## Prerequisites
Ensure you have the following installed on your **Ubuntu** machine:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install curl wget nano vim net-tools unzip -y
sudo apt install openssh-server
sudo apt install python3 python3-pip -y
```

Enable **rsyslog**:

```bash
sudo apt install rsyslog -y
sudo systemctl enable rsyslog
sudo systemctl start rsyslog
```

Edit **rsyslog configuration**:

```bash
sudo vim /etc/rsyslog.conf
```

Add the following lines to enable **UDP & TCP logging**:

```
module(load="imudp")
input(type="imudp" port="514")

module(load="imtcp")
input(type="imtcp" port="514")
```

Restart **rsyslog**:

```bash
sudo systemctl restart rsyslog
```

---

## SSH Setup
Generate SSH keys on your **monitoring server**:

```bash
ssh-keygen -t rsa
```

Copy the public key to the **Asterisk server**:

```bash
ssh-copy-id root@192.168.50.92
```

Or, if needed:

```bash
ssh-copy-id -oHostkeyAlgorithms=+ssh-rsa -oPubkeyAcceptedAlgorithms=+ssh-rsa root@192.168.50.92
```

Test the connection:

```bash
ssh -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedAlgorithms=+ssh-rsa root@192.168.50.92
```

---

## Setting Up Telegram Bot
1. Open Telegram and search for `@BotFather`.
2. Start a chat and send `/newbot` to create a new bot.
3. Follow the instructions to set a name and username.
4. Copy the **TOKEN** provided by BotFather.
5. Get your **Chat ID**:
   
   ```bash
   curl -s "https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getUpdates"
   ```

6. Test message sending:
   
   ```bash
   curl -s -X POST "https://api.telegram.org/bot<YOUR_BOT_TOKEN>/sendMessage" -d chat_id=<YOUR_CHAT_ID> -d text="Test message"
   ```

---

## Bash Script
Create a monitoring script:

```bash
#!/bin/bash

ASTERISK_SERVER="192.168.xx.xx"
LOG_FILE="/var/log/trunks_status.log"
TELEGRAM_BOT_TOKEN="<YOUR_BOT_TOKEN>"
CHAT_ID="<YOUR_CHAT_ID>"

status=$(ssh -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedAlgorithms=+ssh-rsa root@$ASTERISK_SERVER "asterisk -rx 'sip show peers'")

echo "[$(date)] Trunks Status:" | sudo tee -a $LOG_FILE
echo "$status" | sudo tee -a $LOG_FILE
echo "----------------------------" | sudo tee -a $LOG_FILE

unreachable_trunks=$(echo "$status" | grep -E "UNREACHABLE|UNKNOWN|Lagged")

if [[ ! -z "$unreachable_trunks" ]]; then
    message="🚨 Trunk Issue Detected!\n$unreachable_trunks"
    curl -s -X POST "https://api.telegram.org/bot$TELEGRAM_BOT_TOKEN/sendMessage" \
        -d chat_id=$CHAT_ID \
        -d text="$message"
fi
```

Make the script executable:

```bash
chmod +x /home/iss/Bash/check_trunks.sh
```

---

## Automating Execution
Use **cron** to run the script **every minute**:

```bash
crontab -e
```

Add this line:

```bash
* * * * * /home/iss/Bash/check_trunks.sh >> /var/log/trunks_status.log 2>&1
```

Restart and check **cron** service:

```bash
sudo systemctl restart cron
sudo systemctl status cron
```

To monitor logs:

```bash
tail -f /var/log/trunks_status.log
```

---
If you Need to 25 sec

## Running Every 25 Seconds
If you need to run the script **every 25 seconds**, use a `while true` loop inside the script instead of **cron**:

```bash
#!/bin/bash

ASTERISK_SERVER="192.168.50.92"
LOG_FILE="/var/log/trunks_status.log"
TELEGRAM_BOT_TOKEN="<YOUR_BOT_TOKEN>"
CHAT_ID="<YOUR_CHAT_ID>"

while true; do
    status=$(ssh -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedAlgorithms=+ssh-rsa root@$ASTERISK_SERVER "asterisk -rx 'sip show peers'")

    echo "[$(date)] Trunks Status:" | sudo tee -a $LOG_FILE
    echo "$status" | sudo tee -a $LOG_FILE
    echo "----------------------------" | sudo tee -a $LOG_FILE

    curl -s -X POST "https://api.telegram.org/bot$TELEGRAM_BOT_TOKEN/sendMessage" \
        -d chat_id=$CHAT_ID \
        -d text=" Trunk Update: $status"

    sleep 25  # Wait 25 seconds before the next check
done
```

To run it in the background:

```bash
nohup /home/iss/Bash/check_trunks.sh &
```

To stop the script:

```bash
pkill -f check_trunks.sh
```

---

## License
This project is open-source and available under the **MIT License**.

## Contributions
Feel free to contribute by submitting a pull request or reporting issues.

---

Enjoy monitoring your trunks efficiently! 

