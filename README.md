# WhatsApp AI Auto-Reply Agent (n8n)
A complete n8n workflow that receives WhatsApp messages via Meta's Cloud API, runs them through an OpenAI-powered LangChain agent with per-user memory, and sends the AI reply back to the user.

What's in the box
whatsapp-ai-agent.json — drop-in n8n workflow (import via Workflows → Import from File)
system-prompt.md — editable system prompt template
Flow
text

Copy
WhatsApp user

   ↓

Meta Cloud API (webhook)

   ↓

n8n Webhook node

   ↓

Code node: parse payload (phone, text, profile name, message type)

   ↓

AI Agent (LangChain conversational)

   ├─ OpenAI Chat Model (gpt-4o-mini by default)

   └─ Window Buffer Memory (10-message context, keyed on phone)

   ↓

HTTP Request: POST to graph.facebook.com/v21.0/{phone_id}/messages

   ↓

Reply arrives in user's WhatsApp
Setup (10 minutes)
1. Meta side
1.
Go to developers.facebook.com → create a Meta App → add the WhatsApp product.
2.
In WhatsApp → API Setup, grab:
Phone number ID (e.g. 123456789012345)
WhatsApp Business Account ID
3.
Generate a permanent access token (System User token — required for prod, the temp one expires in 1h).
4.
In Configuration → Webhook:
Callback URL: https://YOUR-N8N-DOMAIN.com/webhook/whatsapp-in
Verify token: anything (e.g. mysecret123) — not used in this workflow
Subscribe to: messages
2. n8n side
1.
Import the workflow: Workflows → Import from File → pick whatsapp-ai-agent.json.
2.
Add OpenAI credentials: Settings → Credentials → New → OpenAI API → paste your sk-... key.
3.
Edit the OpenAI Chat Model node: pick your OpenAI credential from the dropdown (it auto-fills the credential id).
4.
Edit the HTTP Request node: replace REPLACE_WITH_YOUR_META_ACCESS_TOKEN with the token from step 1.
5.
Test webhook verification: Meta will send a GET to your webhook URL during config — n8n's Webhook node handles the hub.challenge echo automatically once the workflow is activated. Activate the workflow.
6.
Send a test message from your phone to the Meta test number → you should see the AI reply in ~2-4 seconds.
3. Going live
Verify your business in Meta Business Manager (takes 1-3 days).
Move from test number to a real WhatsApp Business number.
Add a template message branch for outbound messages outside the 24h customer service window.
Customization
Change the AI model
Open the OpenAI Chat Model node → switch from gpt-4o-mini to:

gpt-4o — smarter, ~10x cost
gpt-4.1 / gpt-4.1-mini — newest
any other OpenAI model
You can also swap the whole node for Anthropic, Google Gemini, or Ollama (local) — drop-in replacement.

Switch to Postgres memory (persistent, multi-session)
Replace the Window Buffer Memory node with:

Postgres Chat Memory or Redis Chat Memory — survives n8n restarts
Use {{ $json.session_id }} (the phone number) as the session key, same as the current window memory
Add tools to the agent
Drop a tool node (e.g. HTTP Request Tool, Google Sheets Tool, Calculator) anywhere on the canvas and connect it to the AI Agent via the ai_tool output. The agent will auto-call it when relevant.

Examples that work great:

Google Sheets lookup — for FAQ / product catalog
Calendar tool — for booking appointments
CRM lookup — for customer context
Want outbound campaigns?
Duplicate the workflow, change the trigger to Schedule or Manual, and use a template message body in the HTTP Request — pre-approved templates are required by Meta for outbound.

Common gotchas
Problem	Cause	Fix
Webhook returns 404	Workflow not active	Click Activate Workflow toggle
Messages not received	Webhook not subscribed in Meta	Re-check Webhook Fields in Meta dashboard
Reply says "Invalid OAuth"	Wrong/expired Meta token	Use a permanent System User token, not the temp one
AI ignores system prompt	conversationalAgent is old	Switch to toolsAgent for better prompt adherence
Memory doesn't persist	Window buffer is in-memory	Swap for Postgres / Redis memory
Costs spike	gpt-4o + no token cap	Use gpt-4o-mini + maxTokens: 500 (already set)
Cost estimate
At default settings (gpt-4o-mini, 500 max tokens, ~10-message memory):

~$0.0002 per message in + reply
1,000 conversations ≈ $0.20
10,000 conversations ≈ $2.00
WhatsApp Cloud API is free for the first 1,000 service conversations/month, then ~$0.0042 per conversation.

File reference
whatsapp-ai-agent.json — the workflow (import this)
system-prompt.md — drop-in replacement for the AI Agent's system message
