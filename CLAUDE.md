# Slotted — Projektbriefing für Claude

## Was ist Slotted

Einbettbares Buchungssystem als SaaS. Kunden erhalten einen konfigurierbaren Buchungslink (`/book/[slug]`) und ein Embed-Snippet für ihre Website. Keine Kalender-App — reine Infrastruktur für Terminbuchungen.

**Zielgruppe:** Freelancer, kleine Agenturen, Selbstständige — primär DACH-Markt.

**Kern-Differenzierung:**
- iCloud CalDAV nativ (Calendly kann das nicht sauber)
- Google Meet automatisch erstellt
- Embed-first: Widget läuft auf der Kundenwebsite, nicht als Redirect
- DSGVO-konform, Hosting Deutschland

---

## Tech Stack

- **Astro** (statisch, Vercel)
- **Tailwind v4** — via `@tailwindcss/vite`, kein `tailwind.config.js`
- **Inter Variable** — via `@fontsource-variable/inter`
- **TypeScript strict**
- **Backend:** Laravel API (separates Repo, `api.slotted.de`)
- Keine weiteren Libraries ohne Absprache

---

## Projekt-Infos

- **Pfad:** `/Users/benjaminholz/CODE/slotted`
- **Domain:** slotted.de
- **Dev-Port:** 4322 (Moo Studio läuft auf 4321)
- **Backend:** noch nicht angelegt — API-Calls sind Stubs

---

## Architektur

### Routen

```
/                   Marketing-Seite
/signup             Registrierung
/login              Login
/dashboard          Einstellungen (Slug, iCloud, Google, Zeitplan)
/book/[slug]        Öffentliche Buchungsseite (einbettbar via iframe)
```

### Multi-Tenant

- Jeder User hat: `slug`, `settings` (Zeitplan, iCloud-Credentials, Google OAuth), `bookings`
- Slug ist frei wählbar, unique, URL-safe
- Embed-Snippet: `<iframe src="https://slotted.de/book/[slug]" />`

### Kalender-Integrationen

- **iCloud CalDAV** — Busy-Times lesen, Events anlegen/löschen (Logik aus `BEN-terminbuchung.php` portieren)
- **Google Calendar + Meet** — OAuth2, Events mit Meet-Link anlegen

### E-Mail

- Bestätigung an Kunden (mit ICS-Anhang)
- Benachrichtigung an Owner
- Absage per Token-Link
- Mailtexte: konfigurierbar pro User

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

Hell-Modus ist Default (im Gegensatz zu Moo Studio und DENOYSE).

---

## Referenz-Plugin

Buchungslogik vollständig implementiert in:
`/Users/benjaminholz/CODE/plugins/BEN-terminbuchung/BEN-terminbuchung.php`

Bei Portierung immer dort nachschlagen — Slot-Generierung, CalDAV, ICS-Bau, Absage-Flow.

---

## Arbeitsweise

- Keine ungefragten Libraries oder Abstraktionen
- API-Calls als Stubs bis Backend steht
- Dieses Dokument niemals kürzen — nur ergänzen
