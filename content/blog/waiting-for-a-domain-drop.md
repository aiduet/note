+++
title = "Waiting for a domain drop"
date = "2025-01-04"
slug = "waiting-for-a-domain-drop"
tags = ["notes"]
+++

I am not a domain trader. I just keep a few names, like digital souvenirs.

Good names get grabbed in the redemption window. If the old owner does not pay a high fee, the grabber keeps them. Even average names often get registered first, then go to auction or Dan.com. That is annoying.

So: a nice name almost never drops for you.

The ones worth trying are odd country TLDs, like p.ng or v.ps. I saw an `.ee` name on expireddomains.net about to drop. I do not know the exact time, so I wrote a watcher.

```
import time
import requests
from xml.etree import ElementTree

# Configuration parameters
WHOIS_API_KEY = "YOUR_API_KRY"
DOMAIN = "xx.ee"
WHOIS_API_URL = f"https://www.whoisxmlapi.com/whoisserver/WhoisService?apiKey={WHOIS_API_KEY}&domainName={DOMAIN}"

TELEGRAM_BOT_TOKEN = "YOUR_BOT_TOKEN" 
TELEGRAM_CHAT_ID = "YOUR_CHAT_ID" 
TELEGRAM_API_URL = f"https://api.telegram.org/bot{TELEGRAM_BOT_TOKEN}/sendMessage"

# Function: Send message via Telegram
def send_telegram_message(message):
    payload = {
        "chat_id": TELEGRAM_CHAT_ID,
        "text": message,
    }
    try:
        response = requests.post(TELEGRAM_API_URL, json=payload)
        response.raise_for_status()
        print("Telegram notification sent.")
    except requests.exceptions.RequestException as e:
        print(f"Failed to send Telegram message: {e}")

# Function: Check domain status
def check_domain_status():
    try {
        response = requests.get(WHOIS_API_URL)
        response.raise_for_status()
        tree = ElementTree.fromstring(response.content)
        status = tree.findtext(".//status")
        raw_text = tree.findtext(".//rawText")
        data_error = tree.findtext(".//dataError")

        # Check if domain is available for registration
        if data_error == "MISSING_WHOIS_DATA" or "Domain not found" in raw_text or "expired" not in status :
            print(f"Domain {DOMAIN} is now available!")
            send_telegram_message(f"Domain {DOMAIN} is now available!")
            return True
        else:
            print(f"Current domain status: {status}")
    } catch (requests.exceptions.RequestException e) {
        print(f"WHOIS query failed: {e}")
    }
    return False

# Main loop: Check domain status every 30 seconds
def main():
    while True:
        print("Checking domain availability...")
        if check_domain_status():
            break
        time.sleep(30) 

if __name__ == "__main__":
    main()
```

Direct WHOIS is slow. The `.ee` registry is the bottleneck.

[#domain](/blog/?q=domain)
                
                    [#notes](/blog/?q=notes)
                
                    [#whois](/blog/?q=whois)
