---
title: "Cloudflare DNS Updater Bash Script"
author: isaac
date: 2024-02-06 02:00:00 -0700
categories: [Scripting, Cloudflare, Automation]
tags: [Bash Scripting, DNS Record Updates, Cloudflare API, Error Handling]
render_with_liquid: false
toc: true
comments: true
image:
  path: /assets/img/myphotos/Cloudflare-DNS-Updater-Bash-Script.jpg
  alt: Cloudflare DNS Updater Bash Script
---

I actually use this script in an Azure DevOps pipeline as I run a private hosted agent within my home lab. If you're interested in knowing more about this, leave me a comment below. Otherwise, for the one-off record update, please see below. With some small tweaks, you can update other record types besides A records, but I will not cover this at this time. Please see the [`Cloudflare API documentation`](https://developers.cloudflare.com/api/operations/dns-records-for-a-zone-update-dns-record)

This Bash script is designed to automatically update the "A" record for a specified domain name in Cloudflare. "A" records are DNS records that point your domain name to an IP address. This script uses Cloudflare's HTTP API to perform these updates.

Here's what each part of the script does:

1. The `API_TOKEN` is passed to the script as a command line argument. This token is a unique key that allows the script to interact with the Cloudflare API.
2. The current date and time are fetched and saved to the `current_time` variable. This is used later in the script to mark when updates were performed.
3. The current public IP of the machine running the script is retrieved using the Amazon's Check IP service and stored in the `public_ip` variable.
4. Various variables are defined: `ZONE_ID` is the unique identifier for your Cloudflare zone (which usually contains one or more domain names), `RECORD_NAME` is the domain name for which the DNS record is to be updated.
5. The `H1`, `H2`, and `H3` variables are used to store the headers for the HTTP requests made to Cloudflare's API.
6. The script makes a GET request to the Cloudflare API to retrieve all DNS records for the `ZONE_ID`.
7. From the returned DNS records, the `RECORD_ID` for the required `RECORD_NAME` is extracted.
8. If no `RECORD_ID` is found, the script prints an error message and stops execution.
9. The script checks if the current public IP of the machine is different from that in the DNS record.
10. If the IP address is different, the script generates a JSON payload with the new IP address and updates the DNS record through the Cloudflare API.

To use this script, you need to replace the following placeholders:

- Replace `xxxxxxxxxxxxxxxxxxxxxx` with your Cloudflare Zone ID.
- Replace `record.domain.com` with the domain name you wish to update.
- Replace `youremail@address.com` with your email.

To use the script, copy this script to your local machine:

1. Create a new file using any text editor and paste the script into that file.
2. Save the file with an appropriate name, such as `homeservice.greatdomain.com.sh`.

To change permissions of the file so it can run as a script:

```bash
chmod +x homeservice.greatdomain.com.sh
```

The above command makes the script executable. You can execute the script using the command below, ensuring to replace `<your-api-token>` with your actual Cloudflare API token:

```bash
./homeservice.greatdomain.com.sh <your-api-token>
```

You can also find this script on my github: [`Cloudflare-DNS-Updater-Bash`](https://github.com/SpaceTerran/Cloudflare-DNS-Updater-Bash)

```bash
#!/bin/bash

API_TOKEN=$1 # We get the Cloudflare Global API token from the command line

# Get the current date and time
current_time=$(date "+%m.%d.%Y-%H.%M.%S")

# Get the public IP of the target interface
echo "Getting the public IP of the target interface..."
public_ip=$(curl -s http://checkip.amazonaws.com/)
echo "Public IP: $public_ip"

ZONE_ID="xxxxxxxxxxxxxxxxxxxxxx" # Replace this with your Zone ID
RECORD_NAME="record.domain.com" # Replace this with your doamin name, ex:record.domain.com

# Set the headers for the curl command
H1="X-Auth-Email: youremail@address.com" # Replace this with your Cloudflare email address
H2="X-Auth-Key: $API_TOKEN"
H3="Content-Type: application/json"

# Get all DNS records
echo "Getting DNS records..."
response=$(curl -s -X GET "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/dns_records" -H "$H1" -H "$H2" -H "$H3")

# Extract the record id of the record we want to update
RECORD_ID=$(echo $response | jq -r '.result[] | select(.name=="'"$RECORD_NAME"'") | .id')

# check if record id was found
if [ -z "$RECORD_ID" ]; then
    echo "No record found with the name $RECORD_NAME"
    exit 1
fi

# Get the current content of the record
record_content=$(echo $response | jq -r '.result[] | select(.name=="'"$RECORD_NAME"'") | .content')
echo "Current Record Content: $record_content"

# only update the record if the public ip has changed
if [ "$public_ip" != "$record_content" ]; then

    echo "Record ID: $RECORD_ID"

    # Set the data for the curl command
    DATA=$(cat <<EOF
{
  "type": "A",
  "name": "$RECORD_NAME",
  "content": "$public_ip",
  "ttl": 1,
  "proxied": true,
  "comment": "$current_time"
}
EOF
)

    # Execute the curl command to update the record
    echo "Updating the A record in Cloudflare..."
    response=$(curl -s -o /dev/null -w "%{http_code}" -X PUT "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/dns_records/$RECORD_ID" -H "$H1" -H "$H2" -H "$H3" -d "$DATA")

    if [ $response -eq 200 ]; then
        echo "Successfully updated the A record $RECORD_NAME with public IP $public_ip in Cloudflare Zone $ZONE_ID."
    else
        echo "Failed to update the A record $RECORD_NAME in Cloudflare Zone $ZONE_ID. HTTP code: $response"
    fi
else
    echo "The public IP has not changed. No need to update the record."
fi
```