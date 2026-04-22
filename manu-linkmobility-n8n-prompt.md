# Claude Code Prompt: MANU – Link Mobility WhatsApp & SMS Integration via n8n

## Kontext

Du baust die Messaging-Infrastruktur für **MANU** – einen WhatsApp-basierten KI-Assistenten für Unternehmer. MANU empfängt Nachrichten von Nutzern über WhatsApp (primär) und SMS (Fallback), verarbeitet sie, und antwortet über denselben Kanal. Die gesamte Messaging-Schicht läuft über **Link Mobility** als CPaaS-Provider. Die Orchestrierung erfolgt über **n8n** (self-hosted).

---

## Ziel

Erstelle ein vollständiges n8n-Workflow-Setup (als exportierbare JSON-Dateien) mit folgenden Capabilities:

### 1. WhatsApp Inbound (Empfang)
- **Webhook-Node** in n8n, der als Callback-Endpunkt für Link Mobility dient
- Empfängt eingehende WhatsApp-Nachrichten (MO – Mobile Originated)
- Parst das Link Mobility Webhook-Payload korrekt:

```json
{
  "destination": "+43XXXXXXXXXXX",
  "source": "+43YYYYYYYYYY",
  "content": {
    "type": "WHATSAPP",
    "message": {
      "contentType": "TEXT",
      "text": "Nachricht vom User"
    }
  },
  "provider": "whatsapp",
  "timestamp": "2026-04-07T10:30:00Z",
  "messageId": "uniqueMessageId",
  "providerTimestamp": "2026-04-07T10:30:00Z",
  "providerMessageId": "waMessageId123",
  "route": { ... },
  "gateCustomParameters": {},
  "customParameters": {
    "whatsappUserId": "AT.13491208655302741918",
    "whatsappParentUserId": "AT.11815799212886844830"
  }
}
```

- Unterstützt folgende `contentType`-Werte: `TEXT`, `MEDIA`, `INTERACTIVE` (button_reply, list_reply), `REACTION`
- Extrahiert und speichert `whatsappUserId` / `whatsappParentUserId` (BSUID) für zukünftige Nutzung (ab Juni 2026 relevant, da WhatsApp-Usernames MSISDN ersetzen können)

### 2. WhatsApp Outbound (Senden)
- **HTTP Request Node** für POST an Link Mobility WhatsApp API
- Base URL: `https://n-eu.linkmobility.io/whatsapp-message/messages` (oder `c-eu` – als Variable konfigurierbar)
- Auth: Basic Authentication (Base64 von `username:password`)
- Unterstützt folgende Nachrichtentypen:

#### a) Template-Nachrichten (außerhalb 24h-Fenster)
```json
{
  "platformId": "{{PLATFORM_ID}}",
  "platformPartnerId": "{{PLATFORM_PARTNER_ID}}",
  "priority": "NORMAL",
  "eventReportGates": ["{{EVENT_REPORT_GATE_ID}}"],
  "refId": "manu-{{$now.toISO()}}",
  "source": "{{WHATSAPP_BUSINESS_NUMBER}}",
  "destinations": ["{{recipientNumber}}"],
  "messages": [
    {
      "type": "template",
      "template": {
        "name": "{{templateName}}",
        "language": { "code": "de" },
        "components": [
          {
            "type": "body",
            "parameters": [
              { "type": "text", "text": "{{bodyParam1}}" }
            ]
          }
        ]
      }
    }
  ]
}
```

#### b) Text-Nachrichten (innerhalb 24h Customer Care Window)
```json
{
  "platformId": "{{PLATFORM_ID}}",
  "platformPartnerId": "{{PLATFORM_PARTNER_ID}}",
  "priority": "NORMAL",
  "eventReportGates": ["{{EVENT_REPORT_GATE_ID}}"],
  "refId": "manu-{{$now.toISO()}}",
  "source": "{{WHATSAPP_BUSINESS_NUMBER}}",
  "destinations": ["{{recipientNumber}}"],
  "messages": [
    {
      "type": "text",
      "previewUrl": false,
      "text": { "body": "{{responseText}}" }
    }
  ]
}
```

#### c) Interactive Messages (Reply Buttons)
```json
{
  "platformId": "{{PLATFORM_ID}}",
  "platformPartnerId": "{{PLATFORM_PARTNER_ID}}",
  "priority": "NORMAL",
  "eventReportGates": ["{{EVENT_REPORT_GATE_ID}}"],
  "refId": "manu-{{$now.toISO()}}",
  "source": "{{WHATSAPP_BUSINESS_NUMBER}}",
  "destinations": ["{{recipientNumber}}"],
  "messages": [
    {
      "type": "interactive",
      "interactive": {
        "type": "button",
        "body": { "text": "{{bodyText}}" },
        "action": {
          "buttons": [
            { "type": "reply", "reply": { "id": "btn_1", "title": "Option A" } },
            { "type": "reply", "reply": { "id": "btn_2", "title": "Option B" } }
          ]
        }
      }
    }
  ]
}
```

#### d) Document/Image/Audio/Video Messages
Analog zum Text, aber mit `type: "image"` / `"document"` / `"audio"` / `"video"` und entsprechendem Media-Objekt (`link` oder `id`).

### 3. WhatsApp Delivery Reports (DLR)
- Separater **Webhook-Node** für DLR-Callbacks
- Parst:
```json
{
  "messageId": "...",
  "refId": "...",
  "messageType": "text",
  "messageIndex": 0,
  "timestamp": "...",
  "resultCode": 112003,
  "resultDescription": "delivered",
  "customParameters": { ... }
}
```
- Relevante Result Codes:
  - `112002` = Sent to WhatsApp
  - `112003` = Delivered
  - `112006` = Sent by WhatsApp
  - `112007` = Read
  - `112008` = Deleted
  - `112403` = Not delivered
  - `112404` = Invalid contact
- Speichert Status in einer Datenbank/Tabelle (z.B. n8n-interne Variable, Google Sheet, oder Postgres – konfigurierbar)

### 4. Typing Indicator & Read Receipts
- Nach Empfang einer MO-Nachricht: sofort POST an `/mo/status` senden:
```json
{
  "platformId": "{{PLATFORM_ID}}",
  "platformPartnerId": "{{PLATFORM_PARTNER_ID}}",
  "source": "{{WHATSAPP_BUSINESS_NUMBER}}",
  "status": "read",
  "message_id": "{{providerMessageId}}",
  "typing_indicator": { "type": "text" }
}
```
- Base URL gleich wie Messages-Endpoint

### 5. SMS Outbound (Fallback)
- **HTTP Request Node** für POST an Link Mobility SMS API
- Base URL: `https://eu.linkmobility.io/sms/send`
- Auth: Basic Authentication ODER OAuth 2.0 (Token von `POST /auth/token`)
- Payload:
```json
{
  "platformId": "{{PLATFORM_ID}}",
  "platformPartnerId": "{{PLATFORM_PARTNER_ID}}",
  "source": "MANU",
  "sourceTON": "ALPHANUMERIC",
  "destination": "{{recipientNumber}}",
  "userData": "{{smsText}}",
  "platformServiceType": "manu-sms"
}
```
- SMS wird nur gesendet wenn:
  - WhatsApp-Zustellung fehlgeschlagen (DLR resultCode 112403 oder 112404)
  - Oder Nutzer explizit SMS-only ist

### 6. SMS Inbound
- **Webhook-Node** für eingehende SMS (MO)
- Link Mobility liefert an einen konfigurierten Gate-Endpunkt
- Payload-Struktur ähnlich wie WhatsApp MO, aber mit SMS-spezifischen Feldern

### 7. MANU AI-Processing (Platzhalter)
- Nach Empfang einer Nachricht (WhatsApp oder SMS):
  1. Typing Indicator senden
  2. Nachricht an KI-Backend weiterleiten (HTTP Request an Claude API / eigenes Backend – als Platzhalter-Node)
  3. Antwort empfangen
  4. Über den richtigen Kanal zurücksenden (WhatsApp Text wenn innerhalb 24h, Template wenn außerhalb, SMS als Fallback)

---

## Architektur-Anforderungen

### Credentials (als n8n Credentials konfigurieren)
Erstelle folgende Credential-Variablen, die zentral verwaltet werden:

| Variable | Beschreibung |
|---|---|
| `LM_USERNAME` | Link Mobility API Username |
| `LM_PASSWORD` | Link Mobility API Password |
| `LM_PLATFORM_ID` | Platform ID |
| `LM_PLATFORM_PARTNER_ID` | Platform Partner ID |
| `LM_WHATSAPP_BASE_URL` | z.B. `https://n-eu.linkmobility.io/whatsapp-message` |
| `LM_SMS_BASE_URL` | z.B. `https://eu.linkmobility.io/sms` |
| `LM_EVENT_REPORT_GATE_ID` | Gate ID für DLR-Callbacks |
| `LM_WHATSAPP_SOURCE` | WhatsApp Business Nummer (z.B. `+43...`) |
| `MANU_AI_ENDPOINT` | URL des KI-Backends für Antwortgenerierung |
| `MANU_AI_API_KEY` | API Key für KI-Backend |

### n8n-Workflows (als separate JSON-Dateien)

Erstelle **4 Workflows**:

#### Workflow 1: `manu-whatsapp-inbound.json`
- Trigger: Webhook (POST)
- Schritte:
  1. Parse WhatsApp MO Payload
  2. Sende Read Receipt + Typing Indicator
  3. Extrahiere contentType, text/media/interactive
  4. Speichere BSUID-Mapping (source MSISDN ↔ whatsappUserId)
  5. Route an MANU AI Processing (Workflow 4)

#### Workflow 2: `manu-whatsapp-outbound.json`
- Trigger: Webhook (intern, von Workflow 4 aufgerufen)
- Input: `{ recipientNumber, messageType, content, templateName?, templateParams? }`
- Logik:
  1. Prüfe ob `messageType` = template, text, interactive, media
  2. Baue korrekten Link Mobility Request Body
  3. Sende POST an WhatsApp Messages API
  4. Logge messageId und refId

#### Workflow 3: `manu-sms-fallback.json`
- Trigger: Webhook (intern, von Workflow 4 aufgerufen)
- Input: `{ recipientNumber, smsText }`
- Schritte:
  1. Sende SMS via Link Mobility SMS API
  2. Logge Ergebnis

#### Workflow 4: `manu-ai-processing.json`
- Trigger: Webhook (intern, von Workflow 1 aufgerufen)
- Schritte:
  1. Empfange Nachricht + Kontext (Absender, Kanal, contentType)
  2. Rufe KI-Backend auf (HTTP Request an `MANU_AI_ENDPOINT`)
  3. Empfange Antwort
  4. Entscheide: WhatsApp (Text innerhalb 24h, Template außerhalb) oder SMS Fallback
  5. Rufe entsprechenden Outbound-Workflow auf

### Zusatz-Workflow: `manu-dlr-handler.json`
- Trigger: Webhook (POST) – empfängt DLR-Callbacks
- Schritte:
  1. Parse DLR Payload
  2. Aktualisiere Nachrichtenstatus
  3. Bei `112403`/`112404`: Trigger SMS Fallback

---

## Technische Details

### Authentication Header für alle Link Mobility Requests
```
Authorization: Basic {{base64(LM_USERNAME + ":" + LM_PASSWORD)}}
Content-Type: application/json
```

### Error Handling
- Bei HTTP 401/403: Logge Auth-Fehler, keine Retry
- Bei HTTP 500: Retry 3x mit 60s Delay
- Bei WhatsApp-spezifischen Fehlern (112402 Bad Request): Logge Payload für Debugging
- Bei Quota-Überschreitung (SMS Code 106): Warte und retry

### 24h Customer Care Window Logik
- Speichere den Timestamp der letzten eingehenden Nachricht pro User
- Wenn letzte MO < 24h: Sende als Text/Interactive
- Wenn letzte MO > 24h: Sende als Template
- Implementiere dies als Function-Node in n8n

### BSUID-Handling (Zukunftssicher)
- Ab Juni 2026 können WhatsApp-User Usernames aktivieren → MSISDN fällt weg
- Speichere bei jeder MO-Nachricht das Mapping: `MSISDN ↔ whatsappUserId ↔ whatsappParentUserId`
- Nutze BSUID als Fallback-Identifier wenn MSISDN nicht vorhanden
- BSUIDs können ab Mai 2026 auch als `destinations` im Outbound verwendet werden

### Media Upload (optional, für spätere Phase)
- POST an `/media` (multipart/form-data) mit:
  - `source`: WhatsApp Business Nummer
  - `platformId`, `platformPartnerId`
  - `mediaContentType`: z.B. `image/jpeg`
  - `media`: File
- Response: Media ID (String)
- Media ID kann dann in Outbound-Messages als `"id"` statt `"link"` verwendet werden

---

## Ausgabe-Erwartung

Erstelle:

1. **Alle n8n Workflow JSON-Dateien** (5 Stück), vollständig und importierbar
2. **Eine `README.md`** mit:
   - Setup-Anleitung (Credentials, Webhook-URLs bei Link Mobility registrieren)
   - Architekturdiagramm (Mermaid)
   - Testanleitung (curl-Befehle zum Simulieren von MO-Nachrichten)
3. **Eine `.env.example`** mit allen benötigten Umgebungsvariablen
4. **Ein `test/` Verzeichnis** mit:
   - `test-whatsapp-mo.json` – Beispiel MO Payload
   - `test-dlr.json` – Beispiel DLR Payload
   - `test-sms-mo.json` – Beispiel SMS MO Payload
   - `curl-tests.sh` – Shell-Script mit curl-Befehlen zum Testen

---

## Wichtige Constraints

- Alle Workflows müssen **ohne externen Code** funktionieren (nur n8n-native Nodes: Webhook, HTTP Request, Function, IF, Switch, Set, Merge)
- Credentials dürfen **nicht hardcoded** sein – alles über n8n Credentials oder Environment Variables
- Alle Webhooks müssen **HTTPS** sein (n8n muss hinter einem Reverse Proxy mit SSL laufen)
- Keine externen npm-Packages in Function-Nodes – nur vanilla JavaScript
- Die Workflows sollen **idempotent** sein – doppelte Webhook-Calls dürfen keine doppelten Antworten erzeugen (Deduplizierung über `messageId`)
- Logging: Jeder Workflow soll Error-Handling mit try/catch in Function-Nodes haben

---

## Link Mobility API-Referenz (Kurzfassung)

### WhatsApp API
- **Send**: `POST {BASE_URL}/messages`
- **Typing/Read**: `POST {BASE_URL}/mo/status`
- **Upload Media**: `POST {BASE_URL}/media`
- **Templates CRUD**: `PUT/POST/GET/DELETE {BASE_URL}/platformId/{id}/platformPartnerId/{id}/source/{source}/templates`
- Auth: Basic Authentication
- Format: JSON, UTF-8

### SMS API
- **Send**: `POST {BASE_URL}/send` (einzeln) oder `POST {BASE_URL}/send/batch` (batch)
- **Auth Token**: `POST {BASE_URL}/auth/token` (grant_type=client_credentials)
- Auth: Basic Authentication oder OAuth 2.0 Bearer Token
- Format: JSON, UTF-8

### Result Codes WhatsApp
| Code | Bedeutung |
|---|---|
| 112001 | Queued |
| 112002 | Sent to WhatsApp |
| 112003 | Delivered |
| 112006 | Sent by WhatsApp |
| 112007 | Read |
| 112008 | Deleted |
| 112403 | Not delivered |
| 112404 | Invalid contact |
| 112400 | Unauthorized |
| 112402 | Bad request |
| 112500 | Server error |

### Hosts (Firewall-Freigabe für DLR-Callbacks von Link Mobility)
- `n-eu.linkmobility.io` (North EU)
- `c-eu.linkmobility.io` (Central EU)
- `eu.linkmobility.io` (Auto-Failover)
