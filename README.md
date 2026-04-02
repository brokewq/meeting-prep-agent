# Meeting Prep Agent

An n8n workflow that automatically researches anyone who books a call with you and delivers a ready-to-read briefing to your inbox — before the meeting starts.

**Built with:** n8n (self-hosted) · Cal.com · LimaData API · Claude · Gmail

---

## How it works

1. Someone books a meeting on your Cal.com page
2. The workflow enriches their profile using LimaData (via Profile URL or email)
3. Claude generates a structured meeting prep brief
4. The brief is converted to HTML and emailed to you via Gmail

Every booking gets researched. If the person provided their LinkedIn URL, enrichment uses that (1 credit). If they only gave their email, it falls back to email-based lookup (5 credits). No dead ends.

## What the AI brief includes

- Executive overview — who they are in 3-5 sentences
- Current role and company context
- Career trajectory and patterns
- Domain expertise and skills
- Education and credentials
- Recent Profile activity and interests
- Notable details (side projects, publications, volunteer work)

## Cost

| Tool | Cost |
|------|------|
| n8n | Free (self-hosted) |
| Cal.com | Free tier |
| Claude | Anthropic API key required |
| LimaData | 1-5 credits per lookup (free tier available) |
| Gmail | Free |

## Setup

### Prerequisites

- [n8n](https://n8n.io/) installed and running (local or cloud)
- A [Cal.com](https://cal.com/) account with at least one event type
- A [LimaData](https://limadata.com/) API key
- An [Anthropic](https://console.anthropic.com/) API key
- A Gmail account with OAuth configured in n8n

### Step 1 — Import the workflow

Download `Meeting_Prep_Agent.json` and import it into n8n via **Settings → Import Workflow**.

### Step 2 — Configure credentials

Open each of these nodes and add your credentials:

| Node | What to set |
|------|-------------|
| **Enrich via Profile** | Replace `YOUR_LIMADATA_API_KEY` in the header parameters |
| **Enrich via Email** | Same — replace `YOUR_LIMADATA_API_KEY` |
| **Message a model** | Connect your Anthropic API credential |
| **Notify admin** | Connect your Gmail OAuth credential and set your email in the `sendTo` field |

### Step 3 — Set up Cal.com webhook

1. Open the **Webhook** node in n8n and copy the **Production URL**
2. In Cal.com, go to **Settings → Developer → Webhooks → + New Webhook**
3. Paste the URL into **Subscriber URL**
4. Select **Booking created** as the only trigger
5. Click **Create webhook**

> **Important:** If you're running n8n locally, Cal.com can't reach `localhost`. You'll need to expose your instance via [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/) or [ngrok](https://ngrok.com/).
>
> Quick tunnel: `npx cloudflared tunnel --url http://localhost:5678`

### Step 4 — Add booking form fields

In your Cal.com event type, go to **Advanced → Booking Questions** and add these custom fields:

| Label | Identifier | Input type | Required |
|-------|-----------|------------|----------|
| Linkedin | `Linkedin` (capital L) | URL | No |
| Business Website | `business-website` | URL | No |
| Agenda | `agenda` | Long text | No |

The `name` and `email` fields are built into Cal.com by default.

### Step 5 — Activate and test

1. Toggle the workflow **Active** in n8n
2. Update the Cal.com webhook URL from the test URL to the production URL (remove `-test` from the path)
3. Make a test booking on your Cal.com page
4. Check your inbox for the meeting prep email

## Workflow diagram

```
Cal.com Webhook
      ↓
 Route Events (BOOKING_CREATED)
      ↓
 Extract Payload → Map Fields
      ↓
 Route Enrichment
    ↙       ↘
LinkedIn    Email
(1 credit)  (5 credits)
    ↘       ↙
   Edit Bio
      ↓
    Claude
      ↓
 Markdown → HTML
      ↓
 Gmail → You
```

## Customisation

**Different scheduler?** Replace the Webhook + Route Events nodes with a Calendly or Google Calendar trigger. The rest of the workflow stays the same — just make sure you map the booking fields correctly in the Map Needed Fields node.

**Different AI model?** Replace the Claude node with any LLM node in n8n (OpenAI, Gemini, Ollama for fully local). The system prompt in the messages field works with any model.

**Want a CRM?** Add an Airtable, Notion, or HubSpot node after Map Needed Fields to store booking data, and another after the Gmail node to save the meeting brief back.

## Troubleshooting

**Webhook returns 404 on ping test**
- Make sure n8n is running and listening (click "Test workflow" or activate it)
- Check that the full webhook path is in the URL, not just the tunnel domain
- Verify HTTPS — Cal.com rejects HTTP URLs

**LimaData returns empty or partial data**
- The Enrich Person endpoint returns a lightweight profile (name, headline, about, location, education). For full experience/company data, consider using their Person endpoint (`GET /api/v1/person`) instead.

**Claude returns empty response**
- Confirm the user message (text field) is set to pass the enriched data: `={{ JSON.stringify($json) }}`
- Check your API key is valid at [console.anthropic.com](https://console.anthropic.com/)

**Gmail node fails**
- Re-authenticate your Gmail OAuth credential in n8n
- Make sure the `sendTo` field has your actual email, not a blank expression

## License

MIT — use it however you want.
