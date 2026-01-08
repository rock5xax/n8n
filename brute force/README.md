# Automated Brute Force Login Detection and IP Blocking n8n Workflow

## Overview

This n8n workflow provides an automated system for detecting and blocking brute-force login attacks by monitoring server authentication logs. It periodically checks for multiple failed login attempts from the same IP address, and if an attack is detected, it sends alerts, blocks the offending IP via a firewall API, and records the action in a Google Sheet to prevent duplicate alerts.

This workflow is designed to be robust, configurable, and extensible, serving as a powerful, low-cost security automation tool.

## Features

- **Scheduled Monitoring:** Automatically runs every 5 minutes to check for recent threats.
- **Stateful Blocking:** Utilizes a Google Sheet to keep track of already-blocked IPs, preventing redundant alerts and actions for the same IP address.
- **Configurable Thresholds:** Easily set the number of failed attempts that triggers an alert from a central configuration node.
- **Multi-Channel Alerts:** Simultaneously sends detailed alerts to both Slack and Email (Gmail).
- **Automated IP Blocking:** Calls a configurable firewall API (e.g., Cloudflare) to automatically block the attacker's IP address.
- **Centralized Configuration:** All major parameters (API endpoints, channel IDs, thresholds) are managed in a single "Workflow Configuration" node for easy setup.

## How It Works

The workflow executes in the following sequence:

1.  **Trigger:** A schedule trigger kicks off the workflow every 5 minutes.
2.  **Configuration:** A `Set` node loads all necessary configuration variables, such as API keys, alert channels, and thresholds.
3.  **Read Logs:** The workflow connects to your server via SSH and reads the last 1000 lines of the `/var/log/auth.log` file, searching for "Failed password" entries.
4.  **Parse & Count:** A `Code` node processes the log data, counting the number of failed login attempts for each IP address.
5.  **Check Threshold:** The workflow checks if any IP has exceeded the defined `failedLoginThreshold`.
6.  **Check History:** For each suspicious IP, it looks up the "Blocked IPs" Google Sheet to see if that IP has already been processed.
7.  **Act (If New Attack):** If the IP is over the threshold and has **not** been blocked before, the workflow executes four actions in parallel:
    *   Sends an alert to a specified Slack channel.
    *   Sends an email alert.
    *   Calls your firewall's API to block the IP.
    *   Appends a new row to the Google Sheet to record that the IP has been blocked.

## Prerequisites

Before you begin, you will need:

- A running n8n instance.
- A server accessible via SSH, where authentication logs can be read.
- A Google Account to create a Google Sheet for state tracking.
- A Slack workspace and the ability to create app credentials.
- A Gmail account for sending email alerts.
- An account with a firewall provider that has an API for blocking IPs (e.g., Cloudflare).

## Setup Instructions

### Step 1: Create the Google Sheet

1.  Create a new Google Sheet.
2.  Name the first sheet (tab) `Sheet1`.
3.  In the first row, create the following headers in cells A1, B1, and C1:
    *   `ip`
    *   `blockedAt`
    *   `attempts`

### Step 2: Configure n8n Credentials

Go to the "Credentials" section in your n8n instance and create the following credentials. The workflow will use these to authenticate with the different services.

-   **SSH:** To connect to your server.
-   **Google Sheets:** To read and write to your state-tracking sheet.
-   **Slack:** To send alerts to your workspace.
-   **Gmail:** To send email alerts.
-   **HTTP Header Auth:** For your firewall API. Typically, this involves creating a credential with `Authorization` as the name and `Bearer YOUR_API_TOKEN` as the value.

### Step 3: Import the Workflow

1.  Download the `Automated Brute Force Login Detection and IP Blocking.json` file.
2.  In your n8n canvas, select "Import from File" and choose the downloaded JSON file.

### Step 4: Link Credentials and Configure Variables

1.  Open the nodes that have credential errors (e.g., "Read Auth Logs via SSH", "Send Slack Alert", etc.) and select the credentials you created in Step 2 from the dropdown menu.
2.  Open the **"Workflow Configuration"** node. This is where you will set all the essential parameters for the workflow. Fill in the value for each of the following fields:

| Name                  | Description                                                                                             | Example Value                                                                    |
| --------------------- | ------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| `failedLoginThreshold`  | The number of failed attempts to trigger an alert.                                                      | `10`                                                                             |
| `alertEmail`            | The email address to send security alerts to.                                                           | `security-alerts@example.com`                                                    |
| `slackChannel`          | The Slack Channel ID for posting alerts.                                                                | `C012345ABCD`                                                                    |
| `firewallApiUrl`        | The API endpoint for your firewall to block an IP.                                                      | `https://api.cloudflare.com/client/v4/zones/YOUR_ZONE_ID/firewall/access_rules/rules` |
| `blockedIPsSheetId`   | The ID of the Google Sheet you created in Step 1. You can find this in the sheet's URL.                   | `1aBcDeFgHiJkLmNoPqRsTuVwXyZ1234567890`                                           |
| `adminSlackChannel`   | A Slack channel for sending error notifications if the workflow itself fails (optional, for future use). | `C67890EFGH`                                                                     |

### Step 5: Activate the Workflow

Once all credentials and variables are configured, save the workflow and set the "Active" toggle to ON. The workflow will now run automatically on its schedule.
