# Telegram Phone Number Collector (n8n Automation)

An n8n workflow that monitors a Telegram group, automatically extracts phone numbers from text messages, shared contacts, reply threads, and image screenshots (via OCR), then logs them into Google Sheets with smart deduplication.

## Problem

In a family/group Telegram chat, phone numbers get shared informally — typed in different formats, shared as Telegram contacts, mentioned in reply to a request ("does anyone have Anvar's number?"), or sent as a screenshot of a contact card. Manually tracking these in a spreadsheet is tedious and error-prone.

## What it does

1. **Listens** to every message in a Telegram group.
2. **Extracts phone numbers** from three sources:
   - Plain text (handles spaces, dashes, dots, parentheses — e.g. `90 123 45 67`, `+998-90-123-45-67`)
   - Telegram's native "Share Contact" feature
   - Screenshots of contact cards, via OCR (OCR.space API)
3. **Identifies the phone owner** using a priority chain:
   - Shared contact name (most reliable)
   - The text of the message being replied to (e.g. "Anvar's number?" → owner = Anvar)
   - Name mentioned in the message itself
   - Falls back to "unknown" rather than guessing wrong
4. **Deduplicates** against Google Sheets — if the number already exists, it updates the date/sender/context instead of creating a duplicate row.
5. **Normalizes** every number to a consistent `+998XXXXXXXXX` format, validated against real Uzbek mobile operator prefixes.

## Architecture

```
Telegram Trigger
       │
       ├── (photo?) → Download file → OCR API → merge into message
       │
       ▼
  Extraction logic (JS): regex phone matching, owner detection
       │
       ▼
  IF: phone found?
       │
       ▼
  Google Sheets: Get Row(s) (lookup by phone number)
       │
       ▼
  IF: row exists?
   ├── No  → Append Row (full record)
   └── Yes → Update Row (date, sender, context only)
```

## Tech stack

- **n8n** — workflow orchestration (self-hosted via Docker Compose)
- **Telegram Bot API** — message trigger, contact/photo handling
- **Google Sheets API** — data storage, with OAuth2 and lookup-based dedup
- **OCR.space API** — text extraction from image screenshots
- **JavaScript** (n8n Code nodes) — regex-based phone parsing, name-extraction heuristics, Telegram reply-context resolution

## Notable engineering details

- Regex-based phone matcher tolerant of mixed formatting (spaces/dashes/dots), validated against a whitelist of real Uzbek mobile prefixes to reject false positives.
- Owner-name extraction uses Telegram's `reply_to_message` field to resolve *who* a shared number belongs to when it's sent in reply to a request — not just who sent the message.
- Lookup-before-write pattern against Google Sheets to avoid duplicate rows, implemented without a native "upsert" node (two chained IF + Get Row(s) + Append/Update).
- Google Sheets' automatic type coercion (stripping a leading `+` from "number-formatted" cells) was diagnosed and fixed at the data-normalization layer.
- Deployed via Docker Compose, including a custom external task-runner configuration for sandboxed code execution.

## Status

Core text/contact/reply flow: complete and in production use.
OCR screenshot flow: complete, tested against real contact-card screenshots.

## Setup

This repo contains the exported n8n workflow JSON (`workflow.json`). To use it:

1. Import into your own n8n instance.
2. Create a Telegram bot via [@BotFather](https://t.me/BotFather) and add it to your group (disable Privacy Mode).
3. Connect a Google Sheets account with a sheet matching the columns: `Telefon raqami`, `Raqam egasi`, `Kim tomonidan yuborildi`, `Sana`, `Telegram User ID`, `So'rovnoma matni`.
4. Get a free API key from [OCR.space](https://ocr.space/ocrapi) for the screenshot-OCR branch.
5. Update credentials in the workflow and activate it.
