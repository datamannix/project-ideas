# Project Plan: DIY AI Home Assistant (Amazon Echo)

## Overview

Use a spare Amazon Echo as the hardware front-end for a custom AI assistant. The Echo handles wake word detection ("Alexa"), speech-to-text, and text-to-speech natively. You build a custom **Alexa Skill** that intercepts voice commands and routes them to your own AI backend (Claude API or local Ollama), then returns a response for Alexa to speak.

A **Cloudflare Tunnel** exposes your local server to Alexa without opening any ports on your router.

**Estimated total time:** 2–3 weekends  
**Difficulty:** Intermediate

---

## Architecture Overview

```
Echo Mic → Amazon STT → Your Alexa Skill → Cloudflare Tunnel
                                                    ↓
                                         Local Python Server
                                                    ↓
                                           Claude API / Ollama
                                                    ↓
                              Alexa TTS ← Response ← Skill Handler
```

**What Amazon handles:** Wake word, microphone input, speech-to-text, text-to-speech, speaker output  
**What you build:** The skill endpoint, AI query logic, and response formatting  
**What you save vs. original plan:** USB microphone, Whisper STT, Piper TTS, openWakeWord

---

## What You'll Need

- Amazon Echo (any generation)
- Amazon Developer account (free) — developer.amazon.com
- Anthropic API key (if using Claude) — console.anthropic.com
- A machine to run the local server (your media server laptop works perfectly)
- Cloudflare account (free) — cloudflare.com
- A domain name added to Cloudflare (~£8/yr from Cloudflare Registrar, or use a free temporary tunnel URL for development)

---

## Phase 1 — Amazon Developer Setup

- [ ] Sign up at [developer.amazon.com](https://developer.amazon.com) using the **same Amazon account your Echo is registered to** — this is critical, otherwise your skill won't appear on your device during development
- [ ] Go to the **Alexa Developer Console** → Create Skill
  - Skill name: anything you like (e.g. "My Assistant")
  - Primary locale: English (UK)
  - Model: **Custom**
  - Hosting: **Provision your own**
- [ ] Choose the **Start from Scratch** template
- [ ] Note your **Skill ID** from the skill dashboard — you'll need it later

---

## Phase 2 — Design the Skill Interaction Model

This defines how Alexa interprets what you say. You'll create one catch-all intent that passes the full spoken phrase to your backend.

- [ ] In the Alexa Developer Console, go to **Interaction Model → Intents**
- [ ] Create a new custom intent called `AskAssistantIntent`
- [ ] Add a **slot** to it:
  - Slot name: `query`
  - Slot type: `AMAZON.SearchQuery` (captures freeform speech)
- [ ] Add sample utterances that reference the slot:
  ```
  {query}
  ask {query}
  tell me {query}
  what is {query}
  ```
- [ ] Save and **Build the model** (takes ~30 seconds)

After this, saying *"Alexa, ask [your skill name] [anything]"* will capture the full phrase and send it to your backend as `query`.

---

## Phase 3 — Python Environment & Server

- [ ] On your server, install Python 3.11+ if not already present:
  ```bash
  sudo apt update && sudo apt install python3 python3-pip python3-venv -y
  ```
- [ ] Create the project:
  ```bash
  mkdir ~/assistant && cd ~/assistant
  python3 -m venv venv
  source venv/bin/activate
  ```
- [ ] Install dependencies:
  ```bash
  pip install flask ask-sdk-core anthropic python-dotenv
  ```
- [ ] Create a `.env` file:
  ```
  ANTHROPIC_API_KEY=your-key-here
  ALEXA_SKILL_ID=your-skill-id-here
  ```

---

## Phase 4 — Write the Skill Backend

Create `assistant.py`:

```python
from flask import Flask, request
from ask_sdk_core.skill_builder import SkillBuilder
from ask_sdk_core.dispatch_components import AbstractRequestHandler
from ask_sdk_core.utils import is_intent_name, is_request_type
from ask_sdk_webservice_support.webservice_handler import WebserviceSkillHandler
import anthropic
import os
from dotenv import load_dotenv

load_dotenv()

app = Flask(__name__)
sb = SkillBuilder()
client = anthropic.Anthropic(api_key=os.getenv("ANTHROPIC_API_KEY"))

SYSTEM_PROMPT = """You are a home assistant. The user will give you voice commands.
Respond concisely and conversationally — your response will be spoken aloud by Alexa.
Keep answers to 2-3 sentences maximum. Do not use markdown, bullet points, or formatting."""


class LaunchHandler(AbstractRequestHandler):
    def can_handle(self, handler_input):
        return is_request_type("LaunchRequest")(handler_input)

    def handle(self, handler_input):
        return (handler_input.response_builder
                .speak("Assistant ready. What would you like to know?")
                .ask("Go ahead.")
                .response)


class AskAssistantHandler(AbstractRequestHandler):
    def can_handle(self, handler_input):
        return is_intent_name("AskAssistantIntent")(handler_input)

    def handle(self, handler_input):
        slots = handler_input.request_envelope.request.intent.slots
        query = slots.get("query")
        user_input = query.value if query and query.value else "I didn't catch that."

        try:
            message = client.messages.create(
                model="claude-haiku-4-5-20251001",
                max_tokens=300,
                system=SYSTEM_PROMPT,
                messages=[{"role": "user", "content": user_input}]
            )
            response_text = message.content[0].text
        except Exception as e:
            response_text = "Sorry, I had trouble getting a response. Please try again."

        return (handler_input.response_builder
                .speak(response_text)
                .response)


class FallbackHandler(AbstractRequestHandler):
    def can_handle(self, handler_input):
        return is_intent_name("AMAZON.FallbackIntent")(handler_input)

    def handle(self, handler_input):
        return (handler_input.response_builder
                .speak("I'm not sure how to help with that. Try asking me a question.")
                .ask("What would you like to know?")
                .response)


class SessionEndedHandler(AbstractRequestHandler):
    def can_handle(self, handler_input):
        return is_request_type("SessionEndedRequest")(handler_input)

    def handle(self, handler_input):
        return handler_input.response_builder.response


sb.add_request_handler(LaunchHandler())
sb.add_request_handler(AskAssistantHandler())
sb.add_request_handler(FallbackHandler())
sb.add_request_handler(SessionEndedHandler())

skill_handler = WebserviceSkillHandler(skill=sb.create())


@app.route("/", methods=["POST"])
def invoke_skill():
    return skill_handler.verify_request_and_dispatch(
        http_request_headers=dict(request.headers),
        http_request_body=request.data.decode("utf-8")
    )


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

- [ ] Test the server starts without errors:
  ```bash
  source venv/bin/activate
  python assistant.py
  ```

---

## Phase 5 — Expose the Server via Cloudflare Tunnel

Alexa requires a publicly accessible HTTPS endpoint with a valid SSL certificate. Cloudflare Tunnel provides this for free without opening any ports on your router.

- [ ] Sign up at [cloudflare.com](https://cloudflare.com) (free account)
- [ ] Install `cloudflared` on your server:
  ```bash
  curl -L https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg > /dev/null
  echo 'deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared jammy main' | sudo tee /etc/apt/sources.list.d/cloudflared.list
  sudo apt update && sudo apt install cloudflared -y
  ```
- [ ] Authenticate:
  ```bash
  cloudflared tunnel login
  ```
- [ ] Create a tunnel:
  ```bash
  cloudflared tunnel create assistant
  ```
- [ ] Create tunnel config at `~/.cloudflared/config.yml`:
  ```yaml
  tunnel: <your-tunnel-id>
  credentials-file: /home/<user>/.cloudflared/<tunnel-id>.json

  ingress:
    - hostname: assistant.yourdomain.com
      service: http://localhost:5000
    - service: http_status:404
  ```
- [ ] Route DNS:
  ```bash
  cloudflared tunnel route dns assistant assistant.yourdomain.com
  ```
- [ ] Run the tunnel alongside your Flask server:
  ```bash
  cloudflared tunnel run assistant
  ```
- [ ] Confirm the server is reachable:
  ```bash
  curl -X POST https://assistant.yourdomain.com
  # Should return an Alexa-formatted JSON error — not a connection error
  ```

> **No domain yet?** During development you can use a temporary public URL: `cloudflared tunnel --url http://localhost:5000` — it gives you a random `trycloudflare.com` URL you can use in the Alexa console for testing.

---

## Phase 6 — Connect Skill to Your Endpoint

- [ ] In the **Alexa Developer Console**, go to **Endpoint**
- [ ] Select **HTTPS**
- [ ] Enter your Cloudflare URL: `https://assistant.yourdomain.com`
- [ ] For SSL certificate type, select: **My development endpoint has a certificate from a trusted certificate authority** (Cloudflare handles this)
- [ ] Save endpoint
- [ ] Go to the **Test** tab and enable testing for your account
- [ ] Try it in the test console: type *"ask my assistant what's the capital of France"*
- [ ] Confirm a Claude response comes back

---

## Phase 7 — Test on Your Echo

With testing enabled, the skill automatically appears on any Echo registered to your Amazon account.

- [ ] Say: *"Alexa, open [your skill name]"* — it should respond "Assistant ready"
- [ ] Say: *"Alexa, ask [your skill name] [your question]"*
- [ ] Test a range of questions
- [ ] Check your server logs to confirm requests are arriving and Claude is responding:
  ```bash
  sudo journalctl -u assistant -f
  ```

---

## Phase 8 — Run as System Services

- [ ] Create a systemd service for the Flask backend:
  ```bash
  sudo nano /etc/systemd/system/assistant.service
  ```
  ```ini
  [Unit]
  Description=AI Assistant Backend
  After=network.target

  [Service]
  User=<your-username>
  WorkingDirectory=/home/<your-username>/assistant
  ExecStart=/home/<your-username>/assistant/venv/bin/python assistant.py
  Restart=on-failure
  EnvironmentFile=/home/<your-username>/assistant/.env

  [Install]
  WantedBy=multi-user.target
  ```
- [ ] Install the Cloudflare tunnel as a system service:
  ```bash
  sudo cloudflared service install
  ```
- [ ] Enable and start everything:
  ```bash
  sudo systemctl daemon-reload
  sudo systemctl enable --now assistant
  ```
- [ ] Reboot and confirm your Echo still works

---

## Phase 9 — Home Assistant Integration (Optional)

To control smart devices via voice:

- [ ] Install **Home Assistant** (on a Pi or as Docker on your media server)
- [ ] Enable the HA REST API and generate a long-lived access token
- [ ] Add to `.env`:
  ```
  HA_URL=http://homeassistant.local:8123
  HA_TOKEN=your-token-here
  ```
- [ ] Add a control function to `assistant.py`:
  ```python
  def control_device(entity_id: str, action: str):
      import requests
      service = "turn_on" if action == "on" else "turn_off"
      requests.post(
          f"{os.getenv('HA_URL')}/api/services/homeassistant/{service}",
          headers={"Authorization": f"Bearer {os.getenv('HA_TOKEN')}"},
          json={"entity_id": entity_id}
      )
  ```
- [ ] Update your system prompt to list available devices and instruct Claude to output a structured command (e.g. `ACTION:turn_on:light.living_room`) when a control request is detected
- [ ] Parse Claude's response in `AskAssistantHandler` — if it starts with `ACTION:`, call `control_device()` and return a spoken confirmation; otherwise pass the text straight to Alexa

---

## Improvement Ideas (Post-MVP)

- [ ] **Multi-turn conversations** — store the exchange history in the Alexa session and pass it to Claude so it remembers context within a conversation
- [ ] **Web search** — enable the Anthropic web search tool in your API call for queries needing current information
- [ ] **Personalised system prompt** — tune the prompt based on your household's actual needs (names, routines, preferred formats)
- [ ] **Multiple skills** — create a second skill with a different wake phrase routed to a different system prompt (e.g. one for general questions, one for home control only)

---

## Comparison to Original (No-Echo) Plan

| Component | Original Plan | Echo Plan |
|---|---|---|
| Wake word | openWakeWord (software) | Echo built-in |
| Microphone | USB mic required | Echo built-in |
| Speech-to-text | Whisper (local) | Amazon cloud |
| Text-to-speech | Piper (local) | Alexa cloud |
| Speaker | Separate hardware | Echo built-in |
| Privacy | Fully local possible | Voice goes via Amazon |
| Setup complexity | Higher | Lower |
| Extra hardware cost | £20–50 | £0 (Echo already owned) |

---

## Useful Resources

- [Alexa Developer Console](https://developer.amazon.com/alexa/console/ask)
- [ASK Python SDK Docs](https://developer.amazon.com/en-US/docs/alexa/alexa-skills-kit-sdk-for-python/overview.html)
- [Cloudflare Tunnel Docs](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)
- [Anthropic API Docs](https://docs.anthropic.com)
- [Home Assistant REST API](https://developers.home-assistant.io/docs/api/rest/)
