# SECRETS.example.md — Vocily Labs

Copy this file to `SECRETS.md` (gitignored) and fill in the real values.

## n8n
```
N8N_API_KEY=<your-n8n-api-key-jwt>
N8N_URL=https://<your-n8n-domain>
```

## WAHA
```
WAHA_API_KEY=<your-waha-api-key>
WAHA_INTERNAL_URL=http://waha:3000
WAHA_SESSION=default
```

## Gemini (up to 4 rotating keys for free-tier quota)
```
GEMINI_KEY_1=<AIza...>
GEMINI_KEY_2=<AIza...>
GEMINI_KEY_3=<AIza...>
GEMINI_KEY_4=<AIza...>
```

## Google Sheets
```
SHEET_ID=<your-google-sheet-id>
GOOGLE_OAUTH_CRED_ID=<n8n-credential-id>   # reconnect in n8n if >7 days old
```

## WhatsApp numbers
```
LAB_WHATSAPP=918329914036     # WAHA session number (with country code)
STAFF_GROUP_JID=120363411817386712@g.us
```
