# Auto Update

## Overview

The AT-IBA-PA MINIMART POS applications (Admin IMS and Cashier POS) include an integrated Auto Updater built upon the `@tauri-apps/plugin-updater`.

## How It Works

1. **Checking Process**: On startup and periodically, the application queries its designated update endpoint (typically tied to GitHub releases or a custom server) for the `latest-admin.json` or `latest-cashier.json` metadata files.
2. **Status Logic**: The current status is tracked through `UpdaterStage` (idle, checking, downloading, installing, ready, error). UI elements like the `UpdaterStatusBadge` reflect this status.
3. **Downloading**: If an update exists, the background process downloads the binary and prepares the environment for installation.
4. **Resolution**: Once completed, the user is prompted to restart.

## Safe Update Installation
- **Fallback**: Changes will only be fully applied on restart, avoiding runtime crashes.
- **Toggle**: The auto-updater UI can be hidden via specific queries or localStorage overrides during troubleshooting.
