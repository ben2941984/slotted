# Slotted — Projektbriefing für Claude

## Was ist Slotted

Einbettbares Buchungssystem als SaaS. Kunden erhalten einen konfigurierbaren Buchungslink (`/book?s=[slug]`) und ein Embed-Snippet für ihre Website. Keine Kalender-App — reine Infrastruktur für Terminbuchungen.

**Zielgruppe:** Freelancer, kleine Agenturen, Selbstständige — primär DACH-Markt.

**Kern-Differenzierung:**
- iCloud CalDAV nativ (Calendly kann das nicht sauber)
- Google Meet automatisch erstellt
- Embed-first: Widget läuft auf der Kundenwebsite, nicht als Redirect
- DSGVO-konform, Hosting Deutschland

---

## Tech Stack

### Frontend
- **Astro** (statisch, output: static, Vercel)
- **Tailwind v4** — via `@tailwindcss/vite`, kein `tailwind.config.js`
- **Inter Variable** — via `@fontsource-variable/inter`
- **TypeScript strict**
- Keine weiteren Libraries ohne Absprache

### Backend
- **Laravel 13**, SQLite lokal, MySQL/PostgreSQL auf Produktion
- **Laravel Sanctum** — Token-Auth (kein Cookie/Session). Token in `localStorage('sl_token')`.
- **`$middleware->statefulApi()` ist NICHT aktiv** — CSRF-Checks würden Token-Auth brechen
- Pfad lokal: `/Users/benjaminholz/CODE/slotted-api`

---

## Projekt-Infos

- **Frontend-Pfad:** `/Users/benjaminholz/CODE/slotted`
- **Backend-Pfad:** `/Users/benjaminholz/CODE/slotted-api`
- **Domain:** slotted.de
- **Dev-Port Frontend:** 4322
- **Dev-Port Backend:** 8001 (`php artisan serve --port=8001`)
- **ENV:** `PUBLIC_API_URL=http://localhost:8001/api` in `/Users/benjaminholz/CODE/slotted/.env`
- **API-URL im Browser:** `window.SL_API` — wird von `Base.astro` via `define:vars` injiziert

---

## Architektur

### Routen Frontend

```
/                   Landing Page
/signup             Registrierung → POST /api/auth/register
/login              Login → POST /api/auth/login
/dashboard          Buchungsliste + Embed-Snippet
/dashboard/settings Konfiguration (Slug, iCloud, Google, Zeitplan)
/book               Öffentliche Buchungsseite (statisch, Slug aus ?s= Query-Param)
```

**Wichtig:** `/book` ist eine statische Seite. Der Slug kommt aus `window.location.search` (`?s=slug`), nicht aus dem Pfad. Embed-Snippet zeigt `<iframe src="https://slotted.de/book?s=[slug]">`.

### Routen Backend (`/api/...`)

```
POST   auth/register
POST   auth/login
POST   auth/logout                  (auth:sanctum)
GET    auth/me                      (auth:sanctum)

GET    settings                     (auth:sanctum)
PUT    settings                     (auth:sanctum)
POST   settings/caldav-discover     (auth:sanctum)
DELETE caldav/disconnect            (auth:sanctum)
GET    settings/caldav-test         (auth:sanctum, Debug)

GET    bookings                     (auth:sanctum)
POST   bookings/{id}/cancel         (auth:sanctum)

GET    auth/google/redirect         (auth:sanctum, OAuth-Start)
GET    auth/google/callback         (public, OAuth-Callback)
DELETE auth/google/disconnect       (auth:sanctum)

GET    book/{slug}/config           (public)
GET    book/{slug}/slots            (public)
POST   book/{slug}/book             (public)
GET    cancel/{token}               (public)
```

### Multi-Tenant

- Jeder User hat: `slug`, `user_settings` (Zeitplan, iCloud-Credentials, Google OAuth), `bookings`
- Slug ist frei wählbar, unique, URL-safe (`/^[a-z0-9\-]+$/`)

### Datenbankstruktur

**`users`** — Standard Laravel (name, email, password)

**`user_settings`** — 1:1 zu users; slug, slot_minutes, buffer_minutes, days_ahead, booking_lead_hours, workdays (CSV "1,2,3,4,5"), day_start, day_end, blackout_dates (CSV), caldav_user, caldav_pass, caldav_url, google_client_id, google_client_secret, google_access_token, google_refresh_token, google_token_expires, embed_theme (light/dark), embed_layout (default/mini)

**`bookings`** — user_id, start_dt, end_dt, name, email, note, ip, status (confirmed/cancelled), is_google_meet, timezone, google_meet_link, google_event_id, caldav_uid, cancel_token

### Services

- **`SlotService`** — Slot-Generierung: liest workdays, day_start/end, buffer, lead_time, prüft bestehende Bookings + CalDAV-Busy-Times
- **`CalDavService`** — PROPFIND (Autodiscover), REPORT (Busy-Times), PUT (Event anlegen), DELETE (Event löschen); parst alle drei DTSTART-Formate (UTC Z, TZID=..., All-day)
- **`MailService`** — Bestätigung + Owner-Notify mit ICS-Anhang; Absage-Mail; plain PHP `mail()` (Resend als Produktions-Mailer geplant)

---

## Bekannte Pitfalls

- **`is:global` in `book.astro` und `dashboard.astro`** — Inhalte via `innerHTML` injiziert erhalten kein `data-astro-cid-*`. Deshalb `<style is:global>`.
- **`window.SL_API` statt `import.meta.env`** — `import.meta.env` ist nur im Astro-Build verfügbar, nicht in `<script>`-Tags ohne `define:vars`. Base.astro setzt `window.SL_API` via `define:vars` einmalig.
- **Statische Route `/book`** — kein `[slug].astro` möglich ohne SSR-Adapter. Slug kommt aus `?s=` Query-Parameter.
- **`$middleware->statefulApi()` entfernt** — war Ursache für CSRF token mismatch beim Register/Login.
- **Sensitive Fields Guard (Backend)** — `caldav_pass` und `google_client_secret` werden nie mit leerem oder null-Wert überschrieben. Guard in `SettingsController::update()` via `empty($data[$field])`. Laravel wandelt nullable leere Strings manchmal zu null — deshalb `empty()` statt `=== ''`.
- **`has_caldav_pass` / `has_google_secret`** — Backend gibt nur Boolean-Flags zurück, nie die echten Secrets. Frontend zeigt "✓ connected"-Badge statt Klartext.
- **Timezone-Bug** — `date.toISOString()` gibt UTC zurück; in CEST (UTC+2) ist Mitternacht lokal = 22:00 UTC des Vortags → immer `toYmd()` mit lokalen Gettern verwenden, nie `toISOString().slice(0,10)`.
- **Google Event zuerst erstellen** — Google Meet muss vor CalDAV-Event erstellt werden, damit `meet_link` für die CalDAV-Beschreibung verfügbar ist.
- **CalDAV DESCRIPTION** — Zeilenumbrüche als literal `\n` (Backslash-n), nicht als echter Newline. `escapeIcs()` escaped Kommas und Semikolons.

---

## Design Tokens

```
--color-ink:    #0D0D0D
--color-paper:  #F7F7F5
--color-slate:  #6B7280
--color-accent: #2563EB
--color-green:  #16A34A
--color-red:    #DC2626

--font-body: Inter Variable
```

Hell-Modus ist Default.

---

## Referenz-Plugin

Buchungslogik vollständig implementiert in:
`/Users/benjaminholz/CODE/plugins/BEN-terminbuchung/BEN-terminbuchung.php`

Bei Fragen zu Slot-Generierung, CalDAV, ICS-Bau oder Absage-Flow dort nachschlagen.

---

## Abgeschlossene Phasen

- ✅ Astro-Scaffold (Tailwind v4, Inter, TypeScript)
- ✅ Alle Frontend-Routen (Landing, Signup, Login, Dashboard, Settings, Book)
- ✅ Landing Page mit Hero, How-it-works, Feature-Grid, Bottom CTA
- ✅ Dashboard mit Buchungsliste + Embed-Snippet + Copy-Button
- ✅ Buchungsseite — zweispaltiges Layout: Sidebar (Owner-Info, Slot-Zusammenfassung), Kalender, Zeitauswahl, Formular, Erfolgsseite
- ✅ Laravel API — Auth (Sanctum), Settings, Bookings, Public Book-Endpoints
- ✅ Slot-Generierung portiert aus BEN-terminbuchung
- ✅ CalDAV-Integration (Autodiscovery, Busy-Times, Event anlegen/löschen)
- ✅ Mail mit ICS-Anhang (Bestätigung + Owner-Notify + Absage)
- ✅ End-to-End getestet: Buchung, Doppelbuchung (409), Stornierung per Token
- ✅ Google Meet OAuth2 — Client ID + Secret in Settings, Token-Refresh, Meet-Link in Buchungsbestätigung + CalDAV-Notizen
- ✅ iCloud Disconnect-Button (analog zu Google)
- ✅ Sensitive Field Guard — caldav_pass + google_client_secret nie mit leer/null überschreiben
- ✅ has_caldav_pass / has_google_secret Flags — Secrets nie im Klartext zurückgeben
- ✅ Autofill-Fix — secret fields mit autocomplete="new-password" + setTimeout-Clear mit touchedFields-Guard
- ✅ Timezone-Fix — toYmd() mit lokalen Gettern statt toISOString()
- ✅ CalDAV Booking-Details — DESCRIPTION mit Email, Typ, Telefon, Meet-Link
- ✅ Embed-Design — Theme (light/dark) + Layout (default/mini) wählbar in Settings; ?theme= + ?layout= an /book
- ✅ Dark Mode + Mini Layout für Buchungswidget
- ✅ Auto-Save vor Google OAuth-Redirect
- ✅ EN/DE-Toggle auf allen Seiten (Landing, Login, Signup, Dashboard, Settings, Book)
- ✅ Buchungswidget liest ?lang= aus URL-Param, fällt auf Browser-Sprache zurück
- ✅ Embed-Snippet enthält automatisch ?lang=de wenn DE aktiv

## Offen / nächste Schritte

- ⏳ Produktions-Mailer (Resend via SMTP)
- ⏳ Vercel-Deployment (Frontend, slotted.de) + Hetzner (Backend, api.slotted.de)
- ⏳ Embed-Widget auf externer Seite testen

---

## Arbeitsweise

- Keine ungefragten Libraries oder Abstraktionen
- Dieses Dokument niemals kürzen — nur ergänzen
