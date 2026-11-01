# ElevenLabs Post-Call Webhook — Python (FastAPI)
# Install:  pip install fastapi uvicorn python-dotenv
# Run:      uvicorn app:app --host 0.0.0.0 --port 3000
# .env file me:  WEBHOOK_SECRET=wsec_xxxxxxxx

import base64
import hashlib
import hmac
import json
import os
import time

from dotenv import load_dotenv
from fastapi import BackgroundTasks, FastAPI, Request
from fastapi.responses import JSONResponse

load_dotenv()
app = FastAPI()

WEBHOOK_SECRET = os.getenv("WEBHOOK_SECRET", "")
TOLERANCE_SECONDS = 30 * 60  # 30 minute se purana request reject


def verify_signature(raw_body: str, header: str | None, secret: str) -> bool:
    """Header format: t=timestamp,v0=hash"""
    if not header or not secret:
        return False

    parts = dict(p.strip().split("=", 1) for p in header.split(",") if "=" in p)
    timestamp = parts.get("t")
    received = parts.get("v0")
    if not timestamp or not received:
        return False

    # Timestamp check
    if abs(int(time.time()) - int(timestamp)) > TOLERANCE_SECONDS:
        return False

    # HMAC-SHA256 of "timestamp.body"
    expected = hmac.new(
        secret.encode("utf-8"),
        f"{timestamp}.{raw_body}".encode("utf-8"),
        hashlib.sha256,
    ).hexdigest()

    return hmac.compare_digest(expected, received)


def handle_event(event: dict):
    data = event.get("data", {}) or {}
    event_type = event.get("type")

    if event_type == "post_call_transcription":
        print("Call khatam — conversation:", data.get("conversation_id"))
        print("Agent:", data.get("agent_id"))

        for t in data.get("transcript", []) or []:
            print(f"{t.get('role')}: {t.get('message')}")

        analysis = data.get("analysis") or {}
        if analysis:
            print("Summary:", analysis.get("transcript_summary"))
            print("Data collected:", analysis.get("data_collection_results"))

        # TODO: yahan database / Google Sheet / CRM me save karein

    elif event_type == "post_call_audio":
        audio_b64 = data.get("full_audio")
        if audio_b64:
            filename = f"{data.get('conversation_id')}.mp3"
            with open(filename, "wb") as f:
                f.write(base64.b64decode(audio_b64))
            print("Audio saved:", filename)

    elif event_type == "call_initiation_failure":
        print("Call connect nahi hui:", data.get("failure_reason"), data)

    else:
        print("Unknown event type:", event_type)


@app.post("/webhook/elevenlabs")
async def elevenlabs_webhook(request: Request, background: BackgroundTasks):
    raw = (await request.body()).decode("utf-8")
    signature = request.headers.get("elevenlabs-signature")

    if not verify_signature(raw, signature, WEBHOOK_SECRET):
        return JSONResponse({"error": "Invalid signature"}, status_code=401)

    try:
        event = json.loads(raw)
    except json.JSONDecodeError:
        return JSONResponse({"error": "Invalid JSON"}, status_code=400)

    # Pehle 200 return, processing background me
    background.add_task(handle_event, event)
    return {"received": True}


@app.get("/")
def health():
    return {"status": "ElevenLabs webhook server chal raha hai"}
