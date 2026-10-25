+++
title = "Preventing AWS traffic runaway"
date = "2024-03-23"
slug = "preventing-aws-traffic-runaway"
tags = ["notes"]
+++

I use a free AWS box as a reverse proxy. To keep the traffic quota, I wrote a small script. If usage goes over a limit, it pings a Telegram bot.

```json
### config.json
# {
#   "telegram_bot_token": "xxxxx:xxxxxxx",
#   "chat_id": "xxxx",
#   "monthly_limit_gb": 100
# }

import json
import requests
import schedule
import time
import psutil
from datetime import datetime

def load_config():
    with open("config.json", "r") as file:
        return json.load(file)

def log_initial_traffic_usage():
    today = datetime.now()
    if today.day == 1:  # Check if it is the first day of the month
        net_io = psutil.net_io_counters(pernic=True).get('ens5')
        if net_io:
            total_bytes_initial = net_io.bytes_sent + net_io.bytes_recv
            print(f'{total_bytes_initial}')
            with open("traffic_usage.txt", "w") as file:
                file.write(f"{total_bytes_initial}\n")

def get_current_month_usage():
    try:
        with open("traffic_usage.txt", "r") as file:
            initial_bytes = int(file.readline().strip())
            net_io = psutil.net_io_counters(pernic=True).get('ens5')
            if net_io:
                current_total_bytes = net_io.bytes_sent + net_io.bytes_recv
                used_bytes = current_total_bytes - initial_bytes
                used_gb = used_bytes / (1024**3)  # Convert to GB
                return used_gb
    except FileNotFoundError:
        print("Initial traffic usage file not found. Make sure it's generated at the beginning of the month.")
        return None

def send_telegram_message(message, config):
    url = f"https://api.telegram.org/bot{config['telegram_bot_token']}/sendMessage"
    data = {
        "chat_id": config["chat_id"],
        "text": message
    }
    requests.post(url, data=data)

def check_traffic_and_notify(config):
    used_gb = get_current_month_usage()
    print(f'Already used: {used_gb} GB')
    if used_gb is not None and used_gb > config["monthly_limit_gb"]:
        send_telegram_message(f"Attention: Monthly traffic limit exceeded. Used: {used_gb:.2f} GB", config)

config = load_config()

schedule.every().day.at("00:01").do(log_initial_traffic_usage)
schedule.every(15).seconds.do(check_traffic_and_notify, config)

while True:
    schedule.run_pending()
    time.sleep(1)
```
