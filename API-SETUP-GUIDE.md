# API Setup & Configuration Guide

## Overview
This guide walks through setting up all required API integrations for the AI Sales Engine.

---

## 1. n8n Installation

### Self-Hosted (Recommended for Production)

```bash
# Using Docker Compose
version: '3.8'

services:
  n8n:
    image: n8nio/n8n:latest
    restart: always
    ports:
      - "5678:5678"
    environment:
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=admin
      - N8N_BASIC_AUTH_PASSWORD=<SECURE_PASSWORD>
      - N8N_HOST=n8n.yourdomain.com
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - NODE_ENV=production
      - WEBHOOK_URL=https://n8n.yourdomain.com/
      - GENERIC_TIMEZONE=America/New_York
    volumes:
      - n8n_data:/home/node/.n8n
      - ./n8n-local-files:/files

volumes:
  n8n_data:
```

```bash
# Start n8n
docker-compose up -d

# Access at https://n8n.yourdomain.com:5678
```

### Cloud (n8n Cloud)
1. Sign up at https://n8n.io/cloud
2. Create new workspace
3. Note your webhook base URL: `https://[your-workspace].app.n8n.cloud/webhook/`

---

## 2. OpenAI API Setup

### Get API Key
1. Go to https://platform.openai.com/api-keys
2. Click "Create new secret key"
3. Name: `n8n-sales-engine-[CLIENT_NAME]`
4. Copy key: `sk-proj-...`

### Configure in n8n
1. Go to **Credentials** → **Add Credential**
2. Select **OpenAI API**
3. Enter:
   - **API Key**: `sk-proj-...`
   - **Organization ID**: (optional)
4. Save as: `OpenAI API - [CLIENT_NAME]`

### Recommended Model
- **Model**: `gpt-4o` (best balance of speed/quality)
- **Alternative**: `gpt-4o-mini` (faster, cheaper, slightly lower quality)
- **Temperature**: `0.3` (consistent, deterministic responses)

### Cost Estimation
- Average tokens per qualification: ~500 tokens
- Cost per lead: ~$0.01
- 1000 leads/month: ~$10/month

---

## 3. ElevenLabs Voice API Setup

### Get API Key
1. Sign up at https://elevenlabs.io
2. Go to **Profile** → **API Keys**
3. Generate new key
4. Copy: `xi_...`

### Create Conversational AI Agent
1. Go to **Conversational AI** → **Create Agent**
2. Configure:
   - **Name**: `Sales Qualifier - [CLIENT_NAME]`
   - **Voice**: Select professional voice (e.g., "Rachel", "Adam")
   - **First Message**: See `voice-agent-script.md`
   - **System Prompt**: See `voice-agent-script.md`
   - **Language**: English (US)
   - **Max Duration**: 180 seconds
3. **Advanced Settings**:
   - **Webhook URL**: `https://n8n.yourdomain.com/webhook/voice-callback`
   - **Webhook Events**: `conversation.ended`
   - **Metadata Passthrough**: Enabled
4. Copy **Agent ID**: `agent_...`

### Configure in n8n
1. **Credentials** → **Add Credential**
2. Select **HTTP Header Auth**
3. Enter:
   - **Name**: `xi-api-key`
   - **Value**: `xi_...`
4. Save as: `ElevenLabs API`

### Cost Estimation
- Per-minute cost: ~$0.10
- Average call: 2 minutes = $0.20
- 100 hot leads/month: ~$20/month

### Alternative Voice Providers
- **Bland.ai**: More affordable, good quality
- **Vapi.ai**: Developer-friendly, customizable
- **Twilio Voice**: Enterprise-grade, higher cost

---

## 4. WhatsApp Business API Setup

### Option A: Meta WhatsApp Business API (Official)

#### Prerequisites
- Facebook Business Manager account
- Verified business
- Phone number for WhatsApp Business

#### Setup Steps
1. Go to https://business.facebook.com
2. **Business Settings** → **WhatsApp Accounts** → **Add**
3. Follow verification process
4. Create **WhatsApp Business App**
5. Get credentials:
   - **Phone Number ID**: Found in app dashboard
   - **Access Token**: Generate permanent token
   - **Webhook Verify Token**: Create secure random string

#### Create Message Template
1. Go to **WhatsApp Manager** → **Message Templates**
2. Create template: `hot_lead_alert`
3. Category: **Utility**
4. Template:
```
🔥 HOT LEAD ALERT

Name: {{1}}
Company: {{2}}
Summary: {{3}}
Budget: {{4}}

✅ Voice call completed
📞 Call them NOW
```
5. Submit for approval (usually 24-48 hours)

#### Configure in n8n
1. **Credentials** → **Add Credential**
2. Select **HTTP Header Auth**
3. Enter:
   - **Name**: `Authorization`
   - **Value**: `Bearer [ACCESS_TOKEN]`
4. Save as: `WhatsApp API`

### Option B: Twilio WhatsApp (Easier Setup)

1. Sign up at https://twilio.com
2. Get WhatsApp Sandbox number (instant)
3. For production: Request WhatsApp Business approval
4. Get credentials:
   - **Account SID**
   - **Auth Token**
   - **WhatsApp Number**: `whatsapp:+14155238886` (sandbox)

#### Configure in n8n
1. Use Twilio node instead of HTTP Request
2. Enter Account SID and Auth Token

### Cost Estimation
- Meta: $0.005-0.01 per message (varies by country)
- Twilio: $0.005 per message
- 100 alerts/month: ~$1/month

---

## 5. CRM Setup (HubSpot)

### Get API Key
1. Go to https://app.hubspot.com
2. **Settings** → **Integrations** → **Private Apps**
3. Create new app: `n8n Sales Engine`
4. **Scopes** (required):
   - `crm.objects.contacts.read`
   - `crm.objects.contacts.write`
   - `crm.schemas.contacts.read`
5. Generate token
6. Copy: `pat-na1-...`

### Configure Custom Properties
Create these custom contact properties:

```
lead_score: Single-line text
qualification_summary: Multi-line text
budget_indicator: Number (0-10)
urgency_indicator: Number (0-10)
fit_indicator: Number (0-10)
voice_transcript: Multi-line text
voice_call_status: Dropdown (completed|failed|no-answer)
voice_call_duration: Number
budget_range: Single-line text
timeline: Single-line text
decision_maker: Single checkbox
pain_points: Multi-line text
call_sentiment: Dropdown (positive|neutral|negative)
```

### Configure in n8n
1. **Credentials** → **Add Credential**
2. Select **HubSpot API**
3. Enter:
   - **API Key**: `pat-na1-...`
4. Save as: `HubSpot API - [CLIENT_NAME]`

### Alternative: Zoho CRM
1. Go to https://api-console.zoho.com
2. Create **Server-based Application**
3. Generate **Refresh Token**
4. Use Zoho CRM node in n8n

---

## 6. Google Sheets Setup

### Create Service Account
1. Go to https://console.cloud.google.com
2. Create new project: `n8n-sales-engine`
3. Enable **Google Sheets API**
4. **Credentials** → **Create Credentials** → **Service Account**
5. Name: `n8n-service-account`
6. Grant role: **Editor**
7. Create key (JSON format)
8. Download JSON file

### Create Spreadsheet
1. Create new Google Sheet: `AI Sales Engine - [CLIENT_NAME]`
2. Create tabs:
   - `Master_Lead_Log`
   - `Voice_Call_Log`
   - `Cold_Leads_Log`
   - `Error_Log`
   - `Daily_Metrics`
3. Share with service account email: `n8n-service-account@...iam.gserviceaccount.com`
4. Grant **Editor** access
5. Copy Spreadsheet ID from URL: `https://docs.google.com/spreadsheets/d/[SPREADSHEET_ID]/edit`

### Sheet Schemas

#### Master_Lead_Log
```
lead_id | timestamp_received | name | email | phone | source | score | 
intent_summary | voice_call_status | crm_updated | whatsapp_sent | 
response_time_seconds | status | error_log | follow_up_active | 
sequence_type | last_contact | contacted_manually
```

#### Voice_Call_Log
```
lead_id | name | phone | call_initiated_at | call_sid | call_completed_at | 
call_duration | call_status | transcript | sentiment | status
```

#### Cold_Leads_Log
```
timestamp | name | email | source | intent_summary | reasoning
```

#### Error_Log
```
timestamp | lead_id | node_name | error_message | payload
```

#### Daily_Metrics
```
date | total_leads | hot_count | warm_count | cold_count | 
avg_response_time | voice_calls_completed | crm_success_rate | 
whatsapp_delivery_rate
```

### Configure in n8n
1. **Credentials** → **Add Credential**
2. Select **Google Sheets OAuth2 API**
3. Upload JSON key file
4. Save as: `Google Sheets - [CLIENT_NAME]`

---

## 7. Email API Setup (Optional)

### Option A: SendGrid
1. Sign up at https://sendgrid.com
2. Create API key
3. Verify sender email
4. Use HTTP Request node with SendGrid API

### Option B: SMTP (Gmail)
1. Enable 2FA on Gmail
2. Generate App Password
3. Use Email Send node in n8n:
   - **Host**: `smtp.gmail.com`
   - **Port**: `587`
   - **User**: `your-email@gmail.com`
   - **Password**: App password

---

## 8. Environment Variables Configuration

Create `.env` file for each client:

```bash
# Client Identification
CLIENT_ID=acme-corp
CLIENT_NAME=Acme Corporation

# n8n
N8N_WEBHOOK_URL=https://n8n.yourdomain.com/webhook

# Webhook Security
WEBHOOK_AUTH_TOKEN=<generate-secure-random-token>

# OpenAI
OPENAI_API_KEY=sk-proj-...
OPENAI_MODEL=gpt-4o

# ElevenLabs
ELEVENLABS_API_KEY=xi_...
ELEVENLABS_AGENT_ID=agent_...

# WhatsApp
WHATSAPP_PHONE_ID=123456789
WHATSAPP_ACCESS_TOKEN=EAAxxxx...
SALES_REP_PHONE=+1234567890

# CRM
CRM_TYPE=hubspot
HUBSPOT_API_KEY=pat-na1-...
HUBSPOT_PORTAL_ID=12345678

# Google Sheets
GOOGLE_SHEETS_ID=1abc...xyz
GOOGLE_SERVICE_ACCOUNT_EMAIL=n8n-service-account@...iam.gserviceaccount.com

# Email (Optional)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=sales@company.com
SMTP_PASS=app-password

# Product Information (for AI context)
PRODUCT_DESCRIPTION=Enterprise CRM platform for mid-sized B2B companies
PRODUCT_NAME=AcmeCRM
ICP_DESCRIPTION=Mid-sized B2B companies with 50-500 employees in tech/SaaS

# Sales Team
SALES_REP_NAME=Mike Johnson
SALES_REP_EMAIL=mike@company.com
```

### Set in n8n
1. **Settings** → **Environments**
2. Add each variable
3. Reference in workflows: `{{ $env.VARIABLE_NAME }}`

---

## 9. Workflow Import

### Import Workflows
1. Download JSON files:
   - `n8n-workflow-main.json`
   - `n8n-workflow-voice-callback.json`
   - `n8n-workflow-followup.json`
2. In n8n: **Workflows** → **Import from File**
3. Upload each JSON
4. Activate workflows

### Configure Credentials
For each workflow:
1. Open workflow
2. Click nodes with credential warnings
3. Select appropriate credential from dropdown
4. Save workflow

---

## 10. Testing

### Test Webhook
```bash
curl -X POST https://n8n.yourdomain.com/webhook/lead-intake \
  -H "Authorization: Bearer YOUR_WEBHOOK_AUTH_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Test Lead",
    "phone": "+1234567890",
    "email": "test@example.com",
    "company": "Test Corp",
    "source": "website",
    "message": "Need enterprise solution ASAP, budget 100k"
  }'
```

Expected response:
```json
{
  "success": true,
  "lead_id": "1738761000000-abc12345",
  "score": "HOT",
  "message": "Lead received and processed"
}
```

### Verify Flow
1. Check Google Sheets `Master_Lead_Log` for new entry
2. Check CRM for new/updated contact
3. Verify WhatsApp alert received (if HOT)
4. Check ElevenLabs dashboard for call initiated

---

## 11. Monitoring & Maintenance

### Daily Checks
- Review Google Sheets `Daily_Metrics`
- Check `Error_Log` for failures
- Monitor API usage/costs

### Weekly Reviews
- Analyze conversion rates by source
- Review voice call transcripts
- Optimize AI qualification prompt if needed

### Monthly Optimization
- Review and update voice agent script
- Analyze follow-up sequence effectiveness
- Adjust HOT/WARM/COLD scoring thresholds

---

## 12. Security Best Practices

1. **Webhook Authentication**: Always use `WEBHOOK_AUTH_TOKEN`
2. **API Keys**: Store in environment variables, never in workflow JSON
3. **HTTPS Only**: Use SSL certificates for all webhooks
4. **Rate Limiting**: Implement in n8n settings
5. **Data Retention**: Set Google Sheets auto-archive after 90 days
6. **Access Control**: Limit n8n access to authorized personnel only

---

## 13. Troubleshooting

### Common Issues

#### Webhook not receiving data
- Check firewall/security groups
- Verify webhook URL is publicly accessible
- Test with `curl` command

#### AI qualification returning errors
- Check OpenAI API key validity
- Verify API quota not exceeded
- Review prompt format

#### Voice calls not triggering
- Verify ElevenLabs agent ID correct
- Check phone number format (E.164)
- Review ElevenLabs dashboard for errors

#### CRM not updating
- Verify API key has write permissions
- Check custom properties exist
- Review error logs in Google Sheets

#### WhatsApp messages failing
- Confirm template approved by Meta
- Check phone number format
- Verify access token not expired

---

## Support Resources

- **n8n Documentation**: https://docs.n8n.io
- **n8n Community**: https://community.n8n.io
- **OpenAI API Docs**: https://platform.openai.com/docs
- **ElevenLabs Docs**: https://elevenlabs.io/docs
- **WhatsApp Business API**: https://developers.facebook.com/docs/whatsapp
- **HubSpot API**: https://developers.hubspot.com/docs/api/overview
