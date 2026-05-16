# Restaurant Websites – Technische Doku

## System-Architektur

```
Kunde (Browser) ──→ Cloudflare Tunnel ──→ Ionos VPS ──→ Dein Proxmox (Keller)
                                                  │
                                            ┌─────┴──────┐
                                            │   Zoraxy    │
                                            │  (Reverse   │
                                            │   Proxy)    │
                                            └─────┬──────┘
                                                  │
                                     ┌────────────┼────────────┐
                                     │            │            │
                               LXC Basic    LXC Pro      LXC Premium
                               (Caddy)      (Caddy +     (Caddy +
                                             TastyIgn.)   TastyIgn.)
```

**Warum LXC pro Kunde?**
- Isoliert (ein Kunde kann andere nicht beeinflussen)
- Eigene Ressourcen-Limits (RAM/CPU pro LXC)
- Einfaches Backup (PBS snapshot)
- Kunde kann eigene TastyIgniter-Instanz mit eigenem Theme haben
- Einfach klonbar aus Template

## Deployment-Workflow

### 1. Neuen Kunden anlegen

```bash
# Von deinem Proxmox-Host
pct clone 100 <next-id> --full 1
pct set <next-id> --hostname kunde-domain-de
pct resize <next-id> rootfs <size>G
pct start <next-id>

# Im LXC:
pct enter <next-id>
```

Dann im LXC:
```bash
# Caddy einrichten
nano /etc/caddy/Caddyfile

# Domain (über deine DNS-API)
# Bei Cloudflare: CNAME auf deinen Tunnel
# Bei eigener Domain: A-Record auf Ionos VPS IP

# TastyIgniter (nur Pro/Premium)
cd /var/www
composer create-project tastyigniter/tastyigniter
php artisan igniter:up
# Admin-Zugang einrichten
```

### 2. Website deployen

```bash
# LXC hat Git-Zugriff oder wir kopieren per SCP
scp -r /local/website-files/* root@<lxc-ip>:/var/www/html/

# Oder per Git im LXC:
git clone <repo> /var/www/html
```

### 3. Domain-Konfig

**Option A: Subdomain auf restaurant.website**
```
kunde.restaurant.website.  CNAME  dein-vps-oder-tunnel
```
Caddy fängt alle Subdomains automatisch via `*.restaurant.website`.

**Option B: Eigene Domain**
```
restaurant.de.  A  <ionos-vps-ip>
```
Oder Cloudflare Proxy für DDoS-Schutz.

## Caddy-Konfiguration

```caddyfile
# /etc/caddy/Caddyfile

kunde1.de, www.kunde1.de {
    root * /var/www/kunde1
    file_server
    encode gzip
}

kunde2.de {
    root * /var/www/kunde2
    file_server
    encode gzip
    
    # TastyIgniter unter Subpfad
    handle /order/* {
        reverse_proxy localhost:8080
    }
}

# Oder Subdomain
order.kunde2.de {
    reverse_proxy localhost:8080
}
```

Caddy macht SSL automatisch. Kein Zertifikats-Management nötig.

## TastyIgniter – Important Notes

### Lizenz
- **Core:** MIT – frei, kein Source-Code rausgeben
- **Extensions:** Paid ($49-149/Jahr), Regular License = eine Installation
- **Multivendor ($149/Jahr):** Ein System, viele Restaurants, aber NUR Brand Colors (keine eigenen Themes)
- **Single-Tenant:** Jeder Kunde kriegt eigenes LXC mit eigenem TastyIgniter → eigenes Theme möglich

### Fazit fürs Reselling
| Szenario | Lizenz-Kosten | Themes |
|----------|--------------|--------|
| 1 Kunde, Basic (nur HTML) | **€0** | ✅ Unique |
| 1 Kunde, Pro (HTML + TastyIgniter) | $49/Jahr (Delivery) | ✅ Unique pro LXC |
| 10 Kunden, Pro einzeln | $490/Jahr | ✅ Unique pro LXC |
| 10 Kunden, Multi-Vendor | $149/Jahr | ❌ Nur Brand Colors |

**Empfehlung:** Single-Tenant LXCs. Theme-Freiheit ist das Verkaufsargument.

### Extension-Kosten (pro Instanz, pro Jahr)
- Delivery Management: $49
- Takeaway Management: $49  
- Dine-In Management: $49
- Multi Vendor: $149
- Pos: $49

Gib die Kosten 1:1 an den Kunden weiter. Bei $49 im Jahr = ~€4/Monat – unsichtbar in der Monatsmarge.

## CI/CD – Automatisierung

### Für später: Management-Script

```bash
#!/bin/bash
# manage.sh – Restaurant-Management CLI

# Neues Restaurant anlegen
./manage.sh create kunde1.de --tier basic
./manage.sh create kunde2.de --tier pro --domain kunde2.de

# Status
./manage.sh list
./manage.sh status kunde1.de

# Backup
./manage.sh backup kunde1.de

# Updates
./manage.sh update --all
```

### LXC-Template erstellen

```bash
# Basis-LXC mit allem was TastyIgniter braucht
pct create 100 local:vztmpl/ubuntu-24.04-standard_24.04-1_amd64.tar.zst \
  --storage local-zfs \
  --memory 512 \
  --hostname template-tasty

# Im Template installieren
pct enter 100
apt update && apt install -y php8.3 php8.3-{mysql,curl,xml,mbstring,gd,bcmath} \
  mariadb-server composer caddy unzip
```

Dann kannste für jeden Kunden `pct clone 100 <id>` machen und loslegen.

## Pipeline (NocoDB oder Airtable)

Tabelle: `Restaurants`

| Feld | Typ | Beschreibung |
|------|-----|-------------|
| Name | Text | Restaurant-Name |
| Stadt | Text | Ort |
| Status | Select | `🔍 Gefunden` → `📄 Flyer parsen` → `🛠 MVP` → `👁 Review` → `📤 Angebot` → `✅ Kunde` / `❌ Abgelehnt` |
| Adresse | Text | Für Nominatim |
| Telefon | Text | |
| Website | URL | Falls vorhanden (Konkurrenz-Check) |
| Flyer | Attachment | PDF/Foto |
| Angebot | Select | Basic / Pro / Premium |
| Notizen | LongText | |
| Kontaktiert | Date | |

## USA-Expansion

Dein Kollege vor Ort:
- Akquise bei Restaurants ohne Website
- Er schickt Flyer, ich baue die Seite
- Gleiches System, gleicher Server (Geodistribution über Cloudflare)
- Andere Preise ($), anderes Impressum (Terms + Privacy statt Impressum)
- Kein DSGVO, aber CCPA beachten

## Email-Reselling

Optionen:
1. **MXroute** – ~$10/Jahr für unbegrenzt Mailboxen, resellbar, cPanel
2. **Zoho Mail** – €1/Monat/Mailbox, weiße Schildkröte, DSGVO-konform
3. **Mailcow** – Self-hosted auf deinem Proxmox (Docker), €0 Kosten

Mailcow auf Proxmox = maximale Marge, kein externer Anbieter. Aber: Wartungsaufwand.

**Empfehlung:** MXroute resellen. $10/Jahr = €0,83/Monat. Dem Kunden für €3/Monat verkaufen. 260% Marge.

## AI-Bilder

Tools:
- **Midjourney** – Beste Qualität, aber kein API
- **Flux Pro** (via API) – Gut für Food, ~$0,05/Bild
- **DALL-E 3** (OpenAI API) – $0,04/Bild

Workflow: `"Food photography of [gericht name], restaurant plating, warm lighting, shallow depth of field"`

Wir berechnen pauschal €49 für 8 Bilder. Kosten: ~€0,40. Marge: ~99%.

## Preise (Zusammenfassung)

### Einmalig (Setup)
- Basic: €199 / $249
- Pro: €399 / $499
- Premium: €699 / $899

### Monatlich (Hosting + Wartung)
- Basic: €29 / $39
- Pro: €59 / $79
- Premium: €99 / $129

### Jährliche Kosten (uns)
- Pro Kunde, Pro Tier: $49/Jahr Delivery-Extension ≈ €46 ≈ €3,83/Monat
- Pro Kunde, Premium: $147/Jahr Extensions ≈ €138 ≈ €11,50/Monat
- Server: ~€5/Monat (Ionos VPS) – teilt sich auf alle Kunden
- Domain: ~€10-12/Jahr – 1:1 an Kunden

### Marge pro Kunde (Pro Tier)
- Einnahme: €59/Monat
- Kosten: €3,83 (Extension) + ~€0,50 (Server-Anteil) = ~€4,33
- **Netto: ~€54,67/Monat**
- **Jährlich: ~€656**

Bei 10 Kunden: ~€6.560/Jahr passives Einkommen.
Bei 50 Kunden: ~€32.800/Jahr.

## Deine Aufgaben als Anwendungsentwickler

1. **LXC-Template bauen** – Einmalig
2. **TastyIgniter testen** – Theme-System checken, ob es unseren Anforderungen genügt
3. **Caddy-Konfig automatisieren** – DNS-API + Caddy reload per Script
4. **Oder: TastyIgniter forken** (MIT) – Wenn wir langfristig keine Extensions zahlen wollen

## Meine Aufgaben (AI/Hermes)

1. **Flyer parsen** – Design & Menü extrahieren
2. **Website bauen** – Unique HTML/CSS/JS nach Flyer-Design
3. **Design-Tokens definieren** – Für konsistentes Branding
4. **TastyIgniter konfigurieren** – Menü-Daten importieren, Theme anpassen
5. **Sales-Seite erstellen** – Angebotsseite für Neukunden

## Quickstart für Neukunden

1. Kunde schickt Flyer/Menü (PDF, Foto, Link)
2. Ich parse: Restaurant-Daten, Menü, Design-Farben/-Fonts
3. Ich baue die HTML-Website (1-2 Tage)
4. Ich deploye auf deinem Server (Preview-Link)
5. Du prüfst, gibst Feedback, ich passe an
6. Kunde bekommt Link, Zahlung läuft

Für Pro/Premium:
7. Ich richte TastyIgniter im LXC ein
8. Menu-Daten werden importiert
9. Bestellsystem getestet

---

*Stand: Mai 2026*
