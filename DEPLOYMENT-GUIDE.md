# Deployment & Operations Guide

## Pre-Deployment Checklist

### Infrastructure
- [ ] n8n instance deployed (self-hosted or cloud)
- [ ] SSL certificate configured
- [ ] Domain/subdomain configured (e.g., n8n.yourdomain.com)
- [ ] Firewall rules allow webhook traffic
- [ ] Backup strategy in place

### API Accounts
- [ ] OpenAI API key obtained and tested
- [ ] ElevenLabs account created, agent configured
- [ ] WhatsApp Business API approved (or Twilio sandbox ready)
- [ ] CRM API key generated with proper permissions
- [ ] Google Sheets service account created
- [ ] Email API configured (if using)

### Configuration
- [ ] All environment variables set in n8n
- [ ] Google Sheets created with correct schema
- [ ] CRM custom properties created
- [ ] WhatsApp message templates approved
- [ ] Voice agent script uploaded to ElevenLabs

### Testing
- [ ] Webhook endpoint accessible
- [ ] Test lead processed successfully (HOT path)
- [ ] Test lead processed successfully (WARM path)
- [ ] Test lead processed successfully (COLD path)
- [ ] Voice callback received and processed
- [ ] Follow-up sequence triggered correctly
- [ ] Error handling tested (API failures)

---

## Deployment Steps

### 1. Import Workflows

```bash
# Download workflow files
wget https://your-repo.com/n8n-workflow-main.json
wget https://your-repo.com/n8n-workflow-voice-callback.json
wget https://your-repo.com/n8n-workflow-followup.json
```

In n8n:
1. Go to **Workflows** → **Import from File**
2. Import `n8n-workflow-main.json`
3. Import `n8n-workflow-voice-callback.json`
4. Import `n8n-workflow-followup.json`

### 2. Configure Credentials

For each workflow, configure these credentials:

**Main Workflow**:
- Webhook Auth (Header Auth)
- OpenAI API
- ElevenLabs API (HTTP Header Auth)
- Google Sheets OAuth2
- HubSpot API
- WhatsApp API (HTTP Header Auth)

**Voice Callback Workflow**:
- Google Sheets OAuth2
- HubSpot API
- WhatsApp API

**Follow-up Workflow**:
- Google Sheets OAuth2
- WhatsApp API
- Email API (if using)

### 3. Set Environment Variables

In n8n **Settings** → **Environments**:

```
CLIENT_ID=acme-corp
CLIENT_NAME=Acme Corporation
N8N_WEBHOOK_URL=https://n8n.yourdomain.com/webhook
WEBHOOK_AUTH_TOKEN=<secure-token>
OPENAI_API_KEY=sk-proj-...
OPENAI_MODEL=gpt-4o
ELEVENLABS_API_KEY=xi_...
ELEVENLABS_AGENT_ID=agent_...
WHATSAPP_PHONE_ID=123456789
WHATSAPP_ACCESS_TOKEN=EAAxxxx...
SALES_REP_PHONE=+1234567890
CRM_TYPE=hubspot
HUBSPOT_API_KEY=pat-na1-...
HUBSPOT_PORTAL_ID=12345678
GOOGLE_SHEETS_ID=1abc...xyz
PRODUCT_DESCRIPTION=Your product description
PRODUCT_NAME=Your Product
SALES_REP_NAME=Sales Rep Name
```

### 4. Activate Workflows

1. Open each workflow
2. Click **Activate** toggle (top right)
3. Verify status shows "Active"

### 5. Configure Webhook URLs

**Main Webhook**:
```
https://n8n.yourdomain.com/webhook/lead-intake
```

**Voice Callback Webhook**:
```
https://n8n.yourdomain.com/webhook/voice-callback
```

Add voice callback URL to ElevenLabs agent settings.

---

## Integration with Lead Sources

### Website Form

```html
<form id="lead-form">
  <input type="text" name="name" required>
  <input type="email" name="email" required>
  <input type="tel" name="phone" required>
  <input type="text" name="company">
  <textarea name="message"></textarea>
  <button type="submit">Submit</button>
</form>

<script>
document.getElementById('lead-form').addEventListener('submit', async (e) => {
  e.preventDefault();
  
  const formData = new FormData(e.target);
  const data = {
    name: formData.get('name'),
    email: formData.get('email'),
    phone: formData.get('phone'),
    company: formData.get('company'),
    message: formData.get('message'),
    source: 'website',
    campaign: 'homepage-form'
  };
  
  const response = await fetch('https://n8n.yourdomain.com/webhook/lead-intake', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': 'Bearer YOUR_WEBHOOK_AUTH_TOKEN'
    },
    body: JSON.stringify(data)
  });
  
  const result = await response.json();
  console.log('Lead submitted:', result);
  
  // Show success message
  alert('Thank you! We\'ll be in touch shortly.');
});
</script>
```

### Facebook Lead Ads

Use n8n's **Facebook Lead Ads Trigger** node:
1. Add new trigger node
2. Connect Facebook account
3. Select ad account and form
4. Map fields to lead object
5. Connect to "Initialize Lead Object" node

### Zapier Integration

If using Zapier for other integrations:
1. Create Zap: **[Trigger] → Webhooks by Zapier**
2. Action: **POST** to `https://n8n.yourdomain.com/webhook/lead-intake`
3. Map fields to JSON payload

---

## Monitoring & Observability

### Real-Time Dashboard (Google Sheets)

Create a **Dashboard** tab with these formulas:

**Today's Stats**:
```
=COUNTIFS(Master_Lead_Log!A:A, ">="&TODAY())  // Total leads today
=COUNTIFS(Master_Lead_Log!G:G, "HOT", Master_Lead_Log!A:A, ">="&TODAY())  // HOT leads
=COUNTIFS(Master_Lead_Log!G:G, "WARM", Master_Lead_Log!A:A, ">="&TODAY())  // WARM leads
=AVERAGE(Master_Lead_Log!L:L)  // Avg response time
```

**This Week**:
```
=COUNTIFS(Master_Lead_Log!A:A, ">="&TODAY()-7)
```

**Conversion Funnel**:
```
Total Leads → AI Qualified → Voice Contacted → CRM Updated → Demo Scheduled
```

### Error Monitoring

Set up Google Sheets **Conditional Formatting**:
- Error_Log tab: Highlight new errors in red
- Master_Lead_Log: Highlight failed statuses

### Alerts

Create n8n workflow: **Error Alert to Slack/Email**
- Trigger: Schedule every 15 minutes
- Check Error_Log for new entries
- Send notification if errors found

---

## Performance Optimization

### Response Time Optimization

**Target**: < 60 seconds from webhook to voice call

**Bottlenecks**:
1. AI qualification: ~5-10 seconds
2. CRM update: ~2-5 seconds
3. Voice call trigger: ~3-5 seconds

**Optimizations**:
- Use `gpt-4o-mini` instead of `gpt-4o` (faster, cheaper)
- Parallel execution: Update CRM and trigger voice call simultaneously
- Cache common AI responses

### Cost Optimization

**Monthly Cost Breakdown** (100 leads):
- OpenAI: ~$1-2
- ElevenLabs: ~$20-40 (for HOT leads only)
- WhatsApp: ~$1
- CRM API: Usually free tier
- Google Sheets: Free
- **Total**: ~$25-50/month

**Cost Reduction Strategies**:
1. Only trigger voice calls for HOT leads (current design)
2. Use cheaper AI model for initial qualification
3. Batch CRM updates (update once per hour instead of real-time)
4. Use SMS instead of WhatsApp in some regions

### Scalability

**Current Design Capacity**:
- 1000+ leads/day
- Limited by API rate limits, not n8n

**Scaling Strategies**:
1. **Horizontal**: Deploy multiple n8n instances with load balancer
2. **Vertical**: Increase n8n instance resources
3. **Queue-based**: Add Redis queue for high-volume bursts

---

## Multi-Client Deployment

### Client Onboarding Process

1. **Duplicate Master Workflow**
   - In n8n: Click workflow → **Duplicate**
   - Rename: `AI Sales Engine - [CLIENT_NAME]`

2. **Create Client Environment Variables**
   - Prefix with client ID: `ACME_OPENAI_API_KEY`
   - Or use separate n8n instance per client

3. **Set Up Client Resources**
   - Create Google Sheet (from template)
   - Create ElevenLabs agent (clone master)
   - Set up CRM custom properties
   - Configure WhatsApp templates

4. **Test Client Workflow**
   - Send test lead
   - Verify all integrations working
   - Check logging

5. **Go Live**
   - Provide webhook URL to client
   - Train sales team on alerts
   - Monitor first week closely

### Client Isolation

**Option A: Single n8n Instance**
- Use environment variable prefixes
- Separate Google Sheets per client
- Separate CRM accounts
- **Pros**: Easier to manage, lower cost
- **Cons**: Shared resources, potential conflicts

**Option B: Separate n8n Instances**
- Deploy n8n instance per client
- Complete isolation
- **Pros**: Full isolation, client-specific customization
- **Cons**: Higher cost, more maintenance

---

## Maintenance Schedule

### Daily
- [ ] Check Error_Log for new errors
- [ ] Review Daily_Metrics
- [ ] Verify follow-up sequences running

### Weekly
- [ ] Review voice call transcripts
- [ ] Analyze lead quality by source
- [ ] Check API usage and costs
- [ ] Update AI prompts if needed

### Monthly
- [ ] Review and optimize scoring thresholds
- [ ] Analyze conversion rates
- [ ] Update voice agent script based on feedback
- [ ] Review and optimize follow-up sequences
- [ ] Check for n8n updates

### Quarterly
- [ ] Full system audit
- [ ] Review and update documentation
- [ ] Client feedback sessions
- [ ] Cost optimization review

---

## Backup & Disaster Recovery

### Backup Strategy

**n8n Workflows**:
- Export workflows weekly
- Store in Git repository
- Version control all changes

**Google Sheets**:
- Enable version history
- Export to CSV weekly
- Store in cloud storage (S3, Google Drive)

**Environment Variables**:
- Document in secure password manager
- Keep encrypted backup

### Disaster Recovery Plan

**Scenario 1: n8n Instance Failure**
1. Deploy new n8n instance
2. Import workflows from backup
3. Restore environment variables
4. Update webhook URLs
5. Test with sample lead

**Scenario 2: API Key Compromised**
1. Immediately revoke compromised key
2. Generate new key
3. Update in n8n
4. Test all workflows
5. Audit access logs

**Scenario 3: Data Loss**
1. Restore Google Sheets from backup
2. Reconcile with CRM data
3. Verify data integrity
4. Resume operations

---

## Compliance & Legal

### GDPR Compliance
- [ ] Add data retention policy (auto-delete after 90 days)
- [ ] Implement opt-out mechanism
- [ ] Add privacy policy link to forms
- [ ] Log consent for voice calls

### TCPA Compliance (US)
- [ ] Verify consent for voice calls
- [ ] Implement Do Not Call list
- [ ] Add opt-out in all messages
- [ ] Keep consent records

### Data Security
- [ ] Encrypt data in transit (HTTPS)
- [ ] Encrypt data at rest (Google Sheets encryption)
- [ ] Implement access controls
- [ ] Regular security audits

---

## Training Materials

### For Sales Team

**What to Expect**:
1. WhatsApp alert when HOT lead comes in
2. Alert includes: name, company, budget, timeline
3. Full transcript available in CRM
4. Call lead within 1 hour of alert

**How to Stop Follow-up Sequence**:
1. Update CRM field: `Contacted` = Yes
2. Or update Google Sheets: `contacted_manually` = TRUE

### For Marketing Team

**How to Submit Leads**:
1. Use webhook URL: `https://n8n.yourdomain.com/webhook/lead-intake`
2. Required fields: name, email, phone, source
3. Optional: company, message, campaign

**How to Track Performance**:
1. Access Google Sheets dashboard
2. Review Daily_Metrics tab
3. Analyze by source/campaign

---

## Troubleshooting Guide

### Issue: Leads not being qualified

**Check**:
1. Webhook receiving data? (Check n8n execution log)
2. OpenAI API key valid? (Test in credentials)
3. AI response parsing correctly? (Check Parse AI Response node)

**Fix**:
- Review execution log for errors
- Test AI prompt separately
- Check JSON parsing logic

### Issue: Voice calls not triggering

**Check**:
1. ElevenLabs agent ID correct?
2. Phone number in E.164 format?
3. ElevenLabs API key valid?

**Fix**:
- Verify agent ID in ElevenLabs dashboard
- Format phone: `+1234567890`
- Test API key with curl

### Issue: CRM not updating

**Check**:
1. API key has write permissions?
2. Custom properties exist in CRM?
3. Email address valid?

**Fix**:
- Review API key scopes
- Create missing custom properties
- Validate email format

### Issue: Follow-up not sending

**Check**:
1. Schedule trigger active?
2. Leads in Google Sheets have `follow_up_active` = true?
3. WhatsApp/Email credentials valid?

**Fix**:
- Activate schedule trigger
- Check Google Sheets data
- Test credentials

---

## Success Metrics

### KPIs to Track

| Metric | Target | How to Measure |
|--------|--------|----------------|
| Lead Response Time | < 60 seconds | Google Sheets: `response_time_seconds` |
| HOT Lead Conversion | > 30% | CRM: HOT leads → Opportunities |
| Voice Call Connection Rate | > 60% | Voice_Call_Log: `completed` / `total` |
| Follow-up Engagement | > 20% | Track replies to WhatsApp/Email |
| Sales Team Efficiency | +50% | Time saved vs manual qualification |

### Monthly Report Template

```
AI Sales Engine - Monthly Report
Client: [CLIENT_NAME]
Period: [MONTH YEAR]

LEAD VOLUME
- Total Leads: X
- HOT: X (X%)
- WARM: X (X%)
- COLD: X (X%)

PERFORMANCE
- Avg Response Time: X seconds
- Voice Calls Completed: X
- CRM Updates: X
- Follow-ups Sent: X

CONVERSION
- HOT → Demo: X%
- HOT → Closed: X%
- WARM → Demo: X%

COSTS
- OpenAI: $X
- ElevenLabs: $X
- WhatsApp: $X
- Total: $X
- Cost per Lead: $X

INSIGHTS
- Top performing source: X
- Best converting message type: X
- Recommendations: [...]
```

---

## Next Steps After Deployment

1. **Week 1**: Monitor closely, fix any issues
2. **Week 2**: Optimize AI prompts based on results
3. **Week 3**: Analyze conversion data, adjust scoring
4. **Week 4**: Full performance review, client feedback
5. **Month 2+**: Continuous optimization, scale to more clients
