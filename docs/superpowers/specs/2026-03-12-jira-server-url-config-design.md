# Jira Server URL Configuration — Design Spec

**Date:** 2026-03-12
**Status:** Approved

## Problem

The Quickshare extension's popup only recognises Jira Cloud (`*.atlassian.net`) pages. Users on self-hosted Jira Server / Data Center instances see "Not on a Jira issue page." because `popup.js` hardcodes the hostname check to `.atlassian.net`.

## Goal

Let users configure a single Jira Server base URL so the popup works on self-hosted instances, while preserving existing Jira Cloud behaviour with no configuration required.

## Scope

- Popup functionality only (the injected Share button in `content.js` is out of scope).
- Single URL (not multiple).
- Chrome and Firefox MV3.

## Design

### Storage

`chrome.storage.sync` stores one optional key:

```json
{ "jiraServerUrl": "https://jira.company.com" }
```

Absent or empty means Server support is disabled. Jira Cloud continues to work with no stored value.

### Options Page (`options.html` + `options.js`)

A minimal page registered as the extension's options UI:

- A labelled text input pre-populated with the stored URL (if any).
- A Save button that validates and persists the URL.
- A brief inline confirmation ("Saved.") on success, inline error on invalid input.

**Validation:** must be a valid `https://` URL; trailing slash stripped before saving.

### Gear Icon in Popup (`popup.html` + `popup.js`)

A small gear icon (⚙) added to the popup header area. Clicking it calls `chrome.runtime.openOptionsPage()`, which opens `options.html` in a new tab in both Chrome and Firefox.

### Popup Host Check (`popup.js`)

On load, `popup.js` reads `jiraServerUrl` from `chrome.storage.sync`. The `getIssueDataFromTab` function currently rejects any tab whose hostname doesn't end in `.atlassian.net`. It will be updated to also accept a tab whose origin exactly matches the stored `jiraServerUrl` (after normalisation — trailing slash stripped, lowercased).

```
isValidJiraHost(tabUrl, storedUrl):
  return tabUrl.hostname.endsWith(".atlassian.net")
      || (storedUrl && tabUrl.origin === normalise(storedUrl))
```

### Manifest Changes (`manifest.json`)

- Add `"storage"` to `"permissions"`.
- Add `"options_ui": { "page": "options.html", "open_in_tab": true }`.

`open_in_tab: true` ensures consistent tab-based behaviour in both Chrome and Firefox (without it Firefox embeds the options page inside `about:addons`).

## Files Changed

| File | Change |
|------|--------|
| `manifest.json` | Add `storage` permission; add `options_ui` |
| `popup.html` | Add gear icon button |
| `popup.js` | Read storage on load; extract `isValidJiraHost` helper |
| `options.html` | New — URL input form |
| `options.js` | New — load/save logic |

## Out of Scope

- Injecting the Share button (`content.js`) into Jira Server pages.
- Multiple Server URLs.
- HTTP (non-TLS) Jira instances.
