# Jira Server URL Configuration Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add an options page where users configure a Jira Server base URL, and update the popup to recognise that URL alongside Jira Cloud.

**Architecture:** No background script. New `options.html`/`options.js` pages handle storage via `chrome.storage.sync`. The popup is refactored to async/await and reads the stored URL in parallel with the active tab query to decide whether it's on a valid Jira page.

**Tech Stack:** Vanilla JS, Chrome/Firefox MV3, `chrome.storage.sync`, no build step.

---

## Chunk 1: Manifest + Options Page

### Task 1: Update manifest.json

**Files:**
- Modify: `manifest.json`

- [ ] **Step 1: Add `storage` permission and `options_ui` to `manifest.json`**

Open `manifest.json`. The current `"permissions"` array contains `["clipboardWrite", "activeTab"]`. Make these two changes:

```json
{
  "manifest_version": 3,
  "name": "Quickshare",
  "version": "1.0.0",
  "description": "Copy and share Jira issue links in Slack or Markdown format.",
  "permissions": ["clipboardWrite", "activeTab", "storage"],
  "options_ui": {
    "page": "options.html",
    "open_in_tab": true
  },
  "action": {
    "default_popup": "popup.html",
    "default_icon": {
      "16": "icons/icon16.png",
      "48": "icons/icon48.png",
      "128": "icons/icon128.png"
    }
  },
  "content_scripts": [
    {
      "matches": ["https://*.atlassian.net/*"],
      "js": ["content.js"],
      "css": ["content.css"],
      "run_at": "document_idle"
    }
  ],
  "icons": {
    "16": "icons/icon16.png",
    "48": "icons/icon48.png",
    "128": "icons/icon128.png"
  }
}
```

- [ ] **Step 2: Verify the manifest is valid JSON**

```bash
node -e "JSON.parse(require('fs').readFileSync('manifest.json','utf8')); console.log('valid')"
```

Expected output: `valid`

- [ ] **Step 3: Commit**

```bash
git add manifest.json
git commit -m "feat: add storage permission and options_ui to manifest"
```

---

### Task 2: Create options.html

**Files:**
- Create: `options.html`

- [ ] **Step 1: Create `options.html`**

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <title>Quickshare Settings</title>
  <style>
    body {
      width: 420px;
      margin: 32px auto;
      font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
      font-size: 14px;
      color: #172B4D;
    }
    h1 {
      font-size: 18px;
      font-weight: 600;
      margin: 0 0 24px;
      color: #172B4D;
    }
    label {
      display: block;
      font-weight: 500;
      margin-bottom: 6px;
    }
    .hint {
      font-size: 12px;
      color: #6B778C;
      margin-bottom: 8px;
    }
    input[type="url"] {
      width: 100%;
      padding: 7px 10px;
      border: 2px solid #DFE1E6;
      border-radius: 4px;
      font-size: 14px;
      font-family: inherit;
      box-sizing: border-box;
      outline: none;
      transition: border-color 0.15s;
    }
    input[type="url"]:focus {
      border-color: #0052CC;
    }
    .row {
      display: flex;
      align-items: center;
      gap: 10px;
      margin-top: 12px;
    }
    button {
      padding: 7px 16px;
      background: #0052CC;
      color: #fff;
      border: none;
      border-radius: 4px;
      font-size: 14px;
      font-family: inherit;
      cursor: pointer;
    }
    button:hover {
      background: #0065FF;
    }
    .status {
      font-size: 13px;
    }
    .status-ok  { color: #006644; }
    .status-error { color: #DE350B; }
  </style>
</head>
<body>
  <h1>Quickshare Settings</h1>
  <label for="server-url">Jira Server URL</label>
  <div class="hint">Enter the base URL of your self-hosted Jira instance, e.g. https://jira.example.com or https://jira.example.com/jira. Leave blank to disable.</div>
  <input type="url" id="server-url" placeholder="https://jira.example.com">
  <div class="row">
    <button id="save-btn">Save</button>
    <span class="status" id="status"></span>
  </div>
  <script src="options.js"></script>
</body>
</html>
```

- [ ] **Step 2: Verify the file exists and has valid HTML**

```bash
node -e "const s=require('fs').readFileSync('options.html','utf8'); console.log(s.includes('server-url') && s.includes('save-btn') ? 'ok' : 'missing ids')"
```

Expected: `ok`

---

### Task 3: Create options.js

**Files:**
- Create: `options.js`

- [ ] **Step 1: Create `options.js`**

```js
(() => {
  "use strict";

  const urlInput = document.getElementById("server-url");
  const saveBtn  = document.getElementById("save-btn");
  const statusEl = document.getElementById("status");

  function showStatus(text, isError) {
    statusEl.textContent = text;
    statusEl.className = "status" + (isError ? " status-error" : " status-ok");
    setTimeout(() => {
      statusEl.textContent = "";
      statusEl.className = "status";
    }, 2500);
  }

  // Pre-populate with saved value
  chrome.storage.sync.get({ jiraServerUrl: "" }, ({ jiraServerUrl }) => {
    urlInput.value = jiraServerUrl;
  });

  saveBtn.addEventListener("click", () => {
    const raw = urlInput.value.trim();

    // Empty input → delete the stored key
    if (!raw) {
      chrome.storage.sync.remove("jiraServerUrl", () => showStatus("Saved."));
      return;
    }

    // Validate
    let parsed;
    try {
      parsed = new URL(raw);
    } catch {
      showStatus("Invalid URL.", true);
      return;
    }

    if (parsed.protocol !== "https:") {
      showStatus("Only https:// URLs are supported.", true);
      return;
    }

    // Strip trailing slash before storing
    const normalised = parsed.href.replace(/\/$/, "");
    chrome.storage.sync.set({ jiraServerUrl: normalised }, () => showStatus("Saved."));
  });
})();
```

- [ ] **Step 2: Manual verification — load the extension and open the options page**

1. Open Chrome, go to `chrome://extensions`, enable Developer Mode, click "Load unpacked", select the project directory.
2. Right-click the Quickshare extension icon → "Options" (or navigate to the options URL shown in the extension details).
3. Verify the page loads with the URL input and Save button.
4. Type `https://jira.example.com`, click Save → "Saved." appears briefly in green.
5. Reload the options page → input should be pre-filled with `https://jira.example.com`.
6. Clear the input, click Save → "Saved." appears. Reload → input is empty.
7. Type `http://jira.example.com`, click Save → error "Only https:// URLs are supported." in red.
8. Type `not a url`, click Save → error "Invalid URL." in red.

- [ ] **Step 3: Commit**

```bash
git add options.html options.js
git commit -m "feat: add options page for Jira Server URL configuration"
```

---

## Chunk 2: Popup Updates

### Task 4: Add gear icon to popup.html

**Files:**
- Modify: `popup.html`

- [ ] **Step 1: Add gear button and its styles to `popup.html`**

The `.popup-header` div currently has two `<span>` children for the issue key and title. Wrap the header content to position a gear button top-right.

Replace the existing `<style>` block and `<body>` content in `popup.html` with the following complete file:

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <style>
    body {
      width: 240px;
      margin: 0;
      padding: 0;
      font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
      font-size: 14px;
    }
    .popup-header {
      padding: 10px 16px 8px;
      display: flex;
      flex-direction: column;
      gap: 2px;
      position: relative;
    }
    .popup-header-key {
      font-size: 11px;
      font-weight: 600;
      color: #0052CC;
      letter-spacing: 0.02em;
    }
    .popup-header-title {
      font-size: 13px;
      font-weight: 500;
      color: #172B4D;
      line-height: 1.3;
      overflow: hidden;
      text-overflow: ellipsis;
      display: -webkit-box;
      -webkit-line-clamp: 2;
      -webkit-box-orient: vertical;
      padding-right: 20px;
    }
    .popup-settings-btn {
      position: absolute;
      top: 8px;
      right: 10px;
      background: none;
      border: none;
      cursor: pointer;
      font-size: 14px;
      color: #6B778C;
      padding: 2px 4px;
      line-height: 1;
      border-radius: 3px;
    }
    .popup-settings-btn:hover {
      background: #F4F5F7;
      color: #172B4D;
    }
    .popup-divider {
      height: 1px;
      background: #EBECF0;
      margin: 4px 0;
    }
    .popup-item {
      display: block;
      width: 100%;
      padding: 8px 16px;
      border: none;
      background: none;
      color: #172B4D;
      font-size: 14px;
      font-family: inherit;
      text-align: left;
      cursor: pointer;
      box-sizing: border-box;
    }
    .popup-item:hover {
      background: #F4F5F7;
    }
    .popup-actions {
      padding-bottom: 4px;
    }
    .popup-message {
      padding: 12px 16px;
      color: #6B778C;
      font-size: 13px;
      text-align: center;
    }
  </style>
</head>
<body>
  <div id="content">
    <div class="popup-header">
      <span class="popup-header-key" id="issue-key"></span>
      <span class="popup-header-title" id="issue-title"></span>
      <button class="popup-settings-btn" id="settings-btn" aria-label="Open settings">⚙</button>
    </div>
    <div class="popup-divider"></div>
    <div class="popup-actions">
      <button class="popup-item" id="slack-btn">Copy as Slack</button>
      <button class="popup-item" id="markdown-btn">Copy as Markdown</button>
      <button class="popup-item" id="confluence-btn">Copy as Confluence</button>
    </div>
  </div>
  <div id="message" class="popup-message" style="display:none"></div>
  <script src="popup.js"></script>
</body>
</html>
```

- [ ] **Step 2: Verify the file contains the settings button**

```bash
node -e "const s=require('fs').readFileSync('popup.html','utf8'); console.log(s.includes('settings-btn') && s.includes('aria-label') ? 'ok' : 'missing')"
```

Expected: `ok`

---

### Task 5: Refactor popup.js

**Files:**
- Modify: `popup.js`

- [ ] **Step 1: Replace `popup.js` with the updated async/await version**

```js
(() => {
  "use strict";

  const contentEl = document.getElementById("content");
  const messageEl = document.getElementById("message");

  function showMessage(text) {
    contentEl.style.display = "none";
    messageEl.style.display = "block";
    messageEl.textContent = text;
  }

  function isValidJiraHost(tabUrl, storedUrl) {
    if (tabUrl.hostname.endsWith(".atlassian.net")) return true;
    if (storedUrl && storedUrl.trim()) {
      try {
        const base = new URL(storedUrl).href.replace(/\/$/, "");
        return tabUrl.href.startsWith(base + "/");
      } catch {
        return false;
      }
    }
    return false;
  }

  function getIssueDataFromTab(tab, storedUrl) {
    if (!tab || !tab.url) return null;

    let url;
    try { url = new URL(tab.url); } catch { return null; }

    if (!isValidJiraHost(url, storedUrl)) return null;

    // Try /browse/PROJ-123
    const browseMatch = url.pathname.match(/\/browse\/([A-Z][A-Z0-9_]+-\d+)/);
    let issueKey = browseMatch ? browseMatch[1] : null;

    // Fallback: selectedIssue query param (board/modal view)
    if (!issueKey) {
      const selected = url.searchParams.get("selectedIssue");
      if (selected && /^[A-Z][A-Z0-9_]+-\d+$/.test(selected)) {
        issueKey = selected;
      }
    }

    if (!issueKey) return null;

    // Parse title from tab title
    const tabTitle = tab.title || "";
    const titleMatch = tabTitle.match(/^\[?[A-Z][A-Z0-9_]+-\d+\]?\s*[-–]?\s*(.+?)(?:\s*[-–]\s*Jira)?$/i);
    const rawTitle = titleMatch ? titleMatch[1].trim() : tabTitle;
    const title = rawTitle.replace(/^\[?[A-Z][A-Z0-9_]+-\d+\]?\s*[-–:]?\s*/i, "").trim();

    // Canonical URL: Cloud uses origin; Server uses stored base
    const canonicalBase = url.hostname.endsWith(".atlassian.net")
      ? url.origin
      : new URL(storedUrl).href.replace(/\/$/, "");

    return { issueKey, title, url: `${canonicalBase}/browse/${issueKey}` };
  }

  function copyAndClose(format, data) {
    let text;
    if (format === "slack") {
      text = `${data.issueKey}: ${data.url}`;
    } else if (format === "confluence") {
      text = `[${data.issueKey}: ${data.title}|${data.url}]`;
    } else {
      text = `[${data.issueKey}: ${data.title}](${data.url})`;
    }

    navigator.clipboard.writeText(text).then(() => {
      showMessage("Copied!");
      setTimeout(() => window.close(), 600);
    });
  }

  document.getElementById("settings-btn").addEventListener("click", () => {
    chrome.runtime.openOptionsPage();
  });

  (async () => {
    const [tabs, { jiraServerUrl }] = await Promise.all([
      chrome.tabs.query({ active: true, currentWindow: true }),
      chrome.storage.sync.get({ jiraServerUrl: "" })
    ]);

    const data = getIssueDataFromTab(tabs[0], jiraServerUrl);
    if (!data) {
      showMessage("Not on a Jira issue page.");
      return;
    }

    document.getElementById("issue-key").textContent = data.issueKey;
    document.getElementById("issue-title").textContent = data.title;
    document.getElementById("slack-btn").addEventListener("click", () => copyAndClose("slack", data));
    document.getElementById("markdown-btn").addEventListener("click", () => copyAndClose("markdown", data));
    document.getElementById("confluence-btn").addEventListener("click", () => copyAndClose("confluence", data));
  })();
})();
```

- [ ] **Step 2: Manual verification — Jira Cloud still works**

1. Reload the extension in `chrome://extensions`.
2. Navigate to any Jira Cloud issue page (e.g., `https://yourorg.atlassian.net/browse/PROJ-1`).
3. Click the Quickshare icon → popup shows the issue key, title, and three copy buttons.
4. Click "Copy as Slack" → clipboard contains `PROJ-1: https://yourorg.atlassian.net/browse/PROJ-1`.
5. Click ⚙ → options page opens in a new tab.
6. Navigate to a non-Jira page → popup shows "Not on a Jira issue page."

- [ ] **Step 3: Manual verification — Jira Server works with configured URL**

1. Open the options page, enter your Jira Server URL (e.g., `https://jira.example.com`), click Save.
2. Navigate to `https://jira.example.com/browse/PROJ-1` in the browser.
3. Click the Quickshare icon → popup shows the issue key and title.
4. Click "Copy as Slack" → clipboard contains `PROJ-1: https://jira.example.com/browse/PROJ-1`.

- [ ] **Step 4: Manual verification — subpath install**

1. Open the options page, enter `https://jira.example.com/jira`, click Save.
2. Navigate to `https://jira.example.com/jira/browse/PROJ-1`.
3. Click the Quickshare icon → popup shows issue data.
4. Click "Copy as Slack" → clipboard contains `PROJ-1: https://jira.example.com/jira/browse/PROJ-1`.

- [ ] **Step 5: Manual verification — Firefox**

1. Open Firefox, go to `about:debugging#/runtime/this-firefox`, click "Load Temporary Add-on", select `manifest.json`.
2. Repeat steps from Step 2 verification above.
3. Click ⚙ → options page opens in a new tab (Firefox opens options in a tab by default regardless of `open_in_tab`).

- [ ] **Step 6: Commit**

```bash
git add popup.html popup.js
git commit -m "feat: add gear icon and Jira Server URL support to popup"
```
