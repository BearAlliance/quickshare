# Jira Share

Browser extension that copies Jira issue links in Slack or Markdown format.

## Features

- Adds a "Share" button to Jira Cloud issue pages
- Copy as **Slack** format: `<https://tenant.atlassian.net/browse/PROJ-123|PROJ-123: Issue Title>`
- Copy as **Markdown** format: `[PROJ-123: Issue Title](https://tenant.atlassian.net/browse/PROJ-123)`
- Works with full issue views (`/browse/PROJ-123`) and board modal views
- Handles SPA navigation automatically

## Installation

### Chrome

1. Go to `chrome://extensions`
2. Enable **Developer mode** (top-right toggle)
3. Click **Load unpacked**
4. Select this project directory

### Firefox

1. Go to `about:debugging#/runtime/this-firefox`
2. Click **Load Temporary Add-on**
3. Select the `manifest.json` file in this project directory

## Usage

1. Navigate to any Jira Cloud issue page
2. Click the **Share** button in the issue toolbar
3. Choose **Slack** or **Markdown**
4. Paste the formatted link wherever you need it
