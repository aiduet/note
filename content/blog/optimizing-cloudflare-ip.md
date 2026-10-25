+++
title = "Optimizing Cloudflare IP"
date = "2024-02-22"
slug = "optimizing-cloudflare-ip"
tags = ["notes"]
+++

Sites behind Cloudflare often load slowly in China. I first planned to CNAME through Cloudflare SaaS to a faster domain.

Then I saw a NodeSeek user with an API that picks a better Cloudflare IP. I used that API plus the official Cloudflare API, and set the DNS A record myself.

Too many people use the CNAME trick. It breaks often.

I skip the SaaS setup. Plenty of guides exist. The DNS part is enough.

```bash
 curl --request GET \
  --url https://api.cloudflare.com/client/v4/zones/your_zoneid/dns_records \
  --header 'Content-Type: application/json' \
  --header 'X-Auth-Email: your email' \
  --header 'X-Auth-Key: your key'

// config.json

{
  "url": "https://vps789.com/vps/sum/cfIpTop20",
  "email": "",
  "key": "",
  "zone_id": "",
  "dns_id": "",
  "domain": "",
  "telegram_bot_token": "",
  "chat_id": ""
}
```

```python
import requests
import schedule
import ipaddress
import time
import json
import os

cf_networks = [
    "173.245.48.0/20",
    "103.21.244.0/22",
    "103.22.200.0/22",
    "103.31.4.0/22",
    "141.101.64.0/18",
    "108.162.192.0/18",
    "190.93.240.0/20",
    "188.114.96.0/20",
    "197.234.240.0/22",
    "198.41.128.0/17",
    "162.158.0.0/15",
    "104.16.0.0/13",
    "104.24.0.0/14",
    "172.64.0.0/13",
    "131.0.72.0/22"
]

def load_config():
    with open("config.json", "r") as file:
        return json.load(file)

def save_last_ip(ip):
    with open("last_ip.txt", "w") as file:
        file.write(ip)

def load_last_ip():
    if os.path.exists("last_ip.txt"):
        with open("last_ip.txt", "r") as file:
            return file.read()
    else:
        return ""

def is_ip_in_cf(ip, networks):
    ip_address = ipaddress.ip_address(ip)
    
    for network in networks:
        network_obj = ipaddress.ip_network(network, strict=False)
        if ip_address in network_obj:
            return True  
    
    return False

def send_telegram_message(config, message):
    if config['telegram_bot_token'] and config['chat_id']:
        telegram_url = f"https://api.telegram.org/bot{config['telegram_bot_token']}/sendMessage"
        telegram_data = {"chat_id": config['chat_id'], "text": message}
        try:
            requests.post(telegram_url, data=telegram_data)
        except Exception as e:
            print(f"Failed to send Telegram message: {e}")

def get_third_party_data(url, retries=3, delay=120):
    for i in range(retries):
        try:
            response = requests.get(url)
            response.raise_for_status()  
            return response.json()
        except Exception as e:
            print(f"Attempt {i + 1} failed: {e}")
            time.sleep(delay)
    return None

def get_and_validate_data(url, config, retries=3, delay=120):
    data = get_third_party_data(url, retries, delay)
    if not data:
        send_telegram_message(config, "Failed to get data after several retries.")
        return None 
      
    try:
        first_ip = data['data']['good'][0]['ip']
        if not first_ip:
            raise ValueError("IP address is empty.")
    except (KeyError, IndexError, ValueError) as e:
        send_telegram_message(config, f"Data format is invalid or changed: {e}")
        return None 

    if not is_ip_in_cf(first_ip, cf_networks):
        ## send_telegram_message(config, f"The IP address {first_ip} is NOT within the cf")
        print(f"The IP address {first_ip} is NOT within the cf")
        return None

    return first_ip 

def update_a_and_notify():
    config = load_config()
    last_ip = load_last_ip()

    first_ip = get_and_validate_data(config['url'], config)
    if not first_ip:
        return  # Terminate when data acquisition fails or is in an incorrect format

    # Skip update if IP address has not changed
    if first_ip == last_ip:
        print("IP address has not changed, skip update,")
        return
  
    # Update locally stored IP address
    save_last_ip(first_ip)

    # Cloudflare API settings
    headers = {
        "X-Auth-Email": config['email'],
        "X-Auth-Key": config['key'],
        "Content-Type": "application/json"
    }

    cloudflare_api_url = f"https://api.cloudflare.com/client/v4/zones/{config['zone_id']}/dns_records/{config['dns_id']}"
    payload = {
        "type": "A",
        "name": config['domain'],
        "content": first_ip,
        "ttl": 120,
    }

    # Update A record
    update_response = requests.put(cloudflare_api_url, json=payload, headers=headers)
    if update_response.status_code == 200:
        message = f"Successful Domain Renewal {config['domain']} A to {first_ip}"
    else:
        message = f"Failed to update  A record for domain {config['domain']} : {update_response.text}"

    print(message)

# Timed task settings
schedule.every(1).minutes.do(update_a_and_notify)

while True:
    schedule.run_pending()
    time.sleep(1)
```
