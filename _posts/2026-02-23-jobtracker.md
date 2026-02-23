---
title: "I Built a Job Tracker Because Spreadsheets Were Killing Me"
date: 2026-02-23 13:00:00 +0000
categories: [projects, career]
tags: [job-tracker, still-not-hired, electron, vue3, sqlite, desktop-app, job-search, open-source]
description: "still-not-hired is a free, open-source desktop app for tracking job applications with analytics, resume management, and keyword matching. All data stays local."
---

## The Problem With Spreadsheets

At some point during a job search, everyone builds The Spreadsheet.

Company name.
Role.
Date applied.
Status.
Link.
Maybe a notes column if you're feeling optimistic.

It works for two weeks. Then it becomes twenty tabs, conflicting color codes, and a column named "Status2" because you forgot what "Status" was tracking.

I got tired of it. So I built something better.

---

## What I Built

[**still-not-hired**](https://github.com/XhovaniM8/still-not-hired) is a desktop application for tracking job applications. Electron frontend, Vue 3, SQLite on the backend. All data stays local — nothing is sent anywhere.

It does what a spreadsheet does, but actually well.

---

## Features

### Application Tracking

Log applications with company, title, location, salary range, and status. Everything in one place, queryable, sortable, not a color-coded disaster.

### Status Timeline

Track the full pipeline: Applied → Phone Screen → Interview → Offer / Rejected.

You stop losing track of where things are. You also stop telling yourself you "probably" heard back from that company when you absolutely did not.

### Resume Manager

Store multiple resume versions with keyword profiles attached. Useful when you're applying to roles that require slightly different positioning.

### Keyword Matching

Paste a job description. See how your resume's keyword profile compares.

It flags gaps. It won't catch every niche term — the known issue is that the dictionary is predefined, so obscure domain-specific skills may not register unless they're in a skill section or follow recognizable patterns. But it does catch the obvious misses.

### Analytics Dashboard

Sankey diagrams, pie charts, time series.
Rejection funnel.
Application velocity.
Where you're losing interviews.

With enough data, the analytics are actually useful. The TF-IDF keyword analysis needs roughly 10+ saved job descriptions before the results stabilize — with fewer jobs, the frequency analysis gets noisy. That's a known limitation worth being honest about.

### Contact Management

Link contacts to applications. Track who you've talked to, when, and in what context.

### Local Storage

SQLite. On your machine. No accounts, no sync, no cloud.

---

## Download

Releases are available for all three platforms:

| Platform | Download |
|----------|----------|
| macOS | `.dmg` installer or `.zip` |
| Windows | `.exe` installer or portable `.exe` |
| Linux | `.AppImage` or `.deb` |

Get the latest release from the [Releases page](https://github.com/XhovaniM8/still-not-hired/releases).

**A note on warnings:**

- macOS will flag the app on first launch because it is not code-signed. Right-click → Open to bypass it.
- Windows Defender may also flag the installer for the same reason — unsigned binary. It's safe; it just hasn't paid Apple or Microsoft for a certificate.

---

## Build From Source

If you'd rather not trust a binary from a stranger on the internet:

```bash
git clone https://github.com/XhovaniM8/still-not-hired.git
cd still-not-hired
npm install
npm run electron:dev
```

Node.js 18+ required.

---

## Known Limitations

Being honest about what this doesn't do:

- **No PDF import.** Resumes must be pasted as plain text or LaTeX. PDF parsing is not implemented.
- **Keyword extraction is manual.** After pasting resume content, you have to click "Extract keywords from content." It doesn't auto-extract on save.
- **Keyword matching has a predefined dictionary.** Skills outside the built-in list won't be detected unless they're in an explicit skill section or follow CamelCase/alphanumeric patterns.
- **Analytics require volume.** Below ~10 saved jobs, the TF-IDF results aren't reliable.

---

## Why "still-not-hired"

The name is self-explanatory.

I built this during a job search. It's now v1.0.0 and public. The job search is still ongoing. The irony is intentional.

---

## Contributing

The repo is open source under MIT. If you want to contribute, read [CONTRIBUTING.md](https://github.com/XhovaniM8/still-not-hired/blob/main/CONTRIBUTING.md) before opening a PR.

Useful issues include:
- PDF import
- Automatic keyword extraction
- Better dictionary coverage for niche domains

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | Vue 3, Pinia, Vue Router, Tailwind CSS |
| Desktop | Electron |
| Database | better-sqlite3 (SQLite) |
| Charts | Chart.js, D3.js, D3-Sankey |
| Build | Vite, electron-builder |

---

It's free. It's local. It's better than a spreadsheet.

[Download it or build it yourself.](https://github.com/XhovaniM8/still-not-hired/releases)
