---
title: Local Development
description: Running JourneyLoop locally — dev server setup, mockup review, and screenshot tooling.
---

# Local Development

Tooling and conventions for running JourneyLoop on Dobby for local development and UX review.

---

## UX Mockup Server

Mockups live in the JourneyLoop Django app and are served via the local dev server.

**Start the server:**

```bash
cd /home/dobby/Projects/journeyloop-ux && DEBUG=True poetry run python manage.py runserver 8765
```

!!! warning "`DEBUG=True` is required"
    Static files (Tailwind CSS) are not served by Django in production mode. Without `DEBUG=True`, mockup pages will render without styles.

The server binds to `127.0.0.1:8765`. Mockup routes follow the pattern:

```
http://127.0.0.1:8765/mockups/<slug>/
```

---

## Taking Mockup Screenshots

The browser tool cannot screenshot `localhost` URLs. Use **Playwright headless** instead.

**Prerequisites** (one-time install):

```bash
pip3 install playwright --break-system-packages
python3 -m playwright install chromium
```

**Screenshot a mockup:**

```python
python3 -c "
from playwright.sync_api import sync_playwright
with sync_playwright() as p:
    browser = p.chromium.launch(headless=True, args=['--no-sandbox'])
    page = browser.new_page(viewport={'width': 1400, 'height': 900})
    page.goto('http://127.0.0.1:8765/mockups/<slug>/', wait_until='networkidle', timeout=15000)
    page.screenshot(path='/home/dobby/.openclaw/media/screenshots/<slug>.png', full_page=True)
    browser.close()
"
```

Replace `<slug>` with the mockup route name in both the URL and the output path.

**Analyze the screenshot:**

```python
image(images=['/home/dobby/.openclaw/media/screenshots/<slug>.png'], prompt="...")
```

---

## Gotchas

!!! danger "Use `127.0.0.1`, not `localhost`"
    On this machine, `localhost` may resolve to `::1` (IPv6). The Django dev server binds to IPv4 only (`127.0.0.1:8765`), so IPv6 resolution will fail to connect. Always use the explicit IPv4 address.

!!! info "Chromium headless shell, not full Chromium"
    `python3 -m playwright install chromium` installs the headless shell variant — lighter than full Chromium and sufficient for screenshots.

---

## Verified Setup (issue #138)

This workflow was confirmed working on Dobby as of 2026-02-24.

| Component | Version / Detail |
|---|---|
| Dev server | Django via Poetry, port 8765 |
| Screenshot tool | Playwright + Chromium headless (`--no-sandbox`) |
| Viewport | 1400×900 |
| Output path | `/home/dobby/.openclaw/media/screenshots/` |

---

*Last updated: 2026-02-24*
