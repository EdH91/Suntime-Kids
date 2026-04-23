# ☀️ SunTimeKids

**A GPS-linked UV exposure tracker for parents — helping children get the right amount of healthy sunlight every day.**

-----

## What It Does

SunTimeKids combines real-time UV Index data, a child’s skin type, clothing coverage, and sunscreen SPF to calculate how long a child needs to be outside to reach their daily vitamin D goal. When the goal is reached, the app celebrates with a cheerful animation and sound. It also monitors burn risk simultaneously, alerting parents before overexposure occurs.

-----

## Core Features

### Session Tracking

- **“We’re Heading Out!” button** — single tap to start a session, keeping launch frictionless
- **Multi-child selector** — if multiple children are profiled, a pop-up asks which are going out
- **Clothing coverage picker** — cartoon icon selector ranging from swimsuit to full coverage
- **Sunscreen selector** — None / SPF 15 / SPF 30 / SPF 50+
- **Timestamp-based tracking** — session time calculated from start timestamp, not tick-by-tick, so it works correctly when app is backgrounded

### Vitamin D Calculation

The core calculation combines:

- **Real-time UV Index** via Open-Meteo API (free, no API key required)
- **GPS location** — real coordinates from device, defaulting to Essex County NJ
- **Fitzpatrick skin type** (Types I–VI) — visual swatch selector in onboarding
- **Clothing coverage factor** (1.0 down to 0.2)
- **SPF reduction factor** (1.0 down to 0.2)
- **Fitzpatrick base minutes** to 400 IU at UV Index 5: [8, 12, 16, 22, 32, 45]

### Burn Risk Monitoring

- Live burn risk meter displayed during active sessions
- Shifts to primary focus when vitamin D goal is reached and session is extended
- Push notification alert at 80% burn risk threshold
- Calculated from Fitzpatrick burn minutes adjusted for UV Index, clothing, and SPF

### Child Profiles

- Name, age, Fitzpatrick skin type
- Optional profile photo — stored on-device only, never uploaded (COPPA compliant)
- Up to 2 children (expandable)
- Daily vitamin D goals: 400 IU (age 5+), 300 IU (age 2)

### Goal Celebration

- Animated celebration screen on goal completion
- Cheerful visual with confetti animation
- Option to extend session (shifts to burn risk monitoring mode)
- Option to end session and log to history

### Session End Preferences

- **Auto-end** — 60-second countdown prompt on goal completion, auto-closes if no response
- **Keep tracking** — session continues automatically past goal
- Preference set once in Settings, not per session (keeps launch frictionless)

### Dashboard & History

- Weekly bar chart (green = goal hit, blue = partial, grey = no session)
- Monthly trend cards: outdoor time, goals hit, streak, live UV
- Per-child summary with total outdoor time and goals hit
- Recent sessions list with date, time, duration, UV Index, location, goal status

### Location

- GPS-based real location on load
- Inline location modal (no system prompt) for manual city search or coordinate entry
- City geocoding via Open-Meteo geocoding API
- Defaults to Essex County, NJ if GPS unavailable

### Storage

- **IndexedDB** primary (works in iOS home screen PWA)
- **localStorage** secondary fallback
- **Cookie** tertiary fallback
- In-memory fallback for private browsing
- Profiles and sessions persist across app closes

-----

## Technical Stack

|Layer    |Technology                           |
|---------|-------------------------------------|
|Framework|Vanilla HTML/CSS/JS (single file)    |
|UV Data  |Open-Meteo API (`api.open-meteo.com`)|
|Geocoding|Open-Meteo Geocoding API             |
|Storage  |IndexedDB + localStorage + cookies   |
|Fonts    |Nunito + Baloo 2 (Google Fonts)      |
|Hosting  |GitHub Pages (target)                |
|Platform |Mobile-first web app / PWA           |

-----

## File Structure

```
Suntime-Kids/
├── index.html          # Full application (v2.1)
└── README.md           # This file
```

-----

## Version History

|Version|Notes                                                                                                                                                                           |
|-------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|v1.0   |Initial demo — full flow onboarding through dashboard, simulated data                                                                                                           |
|v2.0   |Live UV API, real GPS, timestamp-based background tracking, multi-session persistence (sessionStorage), browser notifications                                                   |
|v2.1   |IndexedDB + cookie storage for iOS PWA persistence, inline location modal replacing prompt(), full-screen mobile layout, iOS PWA meta tags, name bug fix, notification crash fix|

-----

## Known Issues / Next Steps

- [ ] GitHub Pages activation pending (requires desktop browser)
- [ ] Location change button needs live testing on hosted version
- [ ] Storage persistence needs live testing on hosted version
- [ ] Confirm real UV data retrieval on device
- [ ] Add Settings screen (session end preference, notification preferences)
- [ ] Add support for more than 2 children
- [ ] Seasonal UV warnings (low UV months in NJ — Oct through Mar)
- [ ] Annual summary view in dashboard
- [ ] Consider provisional patent for UV calculation algorithm

-----

## Roadmap

|Stage                 |Description                             |Status           |
|----------------------|----------------------------------------|-----------------|
|1 — Validation        |Personal testing with own children      |🔄 In progress    |
|2 — MVP               |Stable hosted version, small focus group|⏳ Pending hosting|
|3 — Revenue proof     |App Store submission, beta              |📋 Planned        |
|4 — Scaling           |Marketing, partnerships                 |📋 Planned        |
|5 — Income replacement|Full commercial operation               |📋 Planned        |

-----

## Science Notes

Vitamin D synthesis is triggered by UV-B radiation (290–315nm). Key variables:

- UV Index is the primary driver — doubles synthesis rate from Index 3 to Index 6
- Cloud cover reduces UV-B by 20–70% depending on thickness
- SPF 30 blocks ~97% of UV-B in lab conditions but ~70–75% in real-world use
- Fitzpatrick Type I synthesises vitamin D ~5x faster than Type VI at equivalent exposure
- October through March in New Jersey: UV-B too weak for meaningful synthesis even on clear days

-----

## Privacy

- No user accounts required
- No data sent to any server
- All child data (profiles, photos, session history) stored locally on device only
- COPPA compliant by design

-----

*Project started April 2026. Built with Claude (Anthropic).*
