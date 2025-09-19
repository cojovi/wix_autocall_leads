now if i fill out the wix form, with my number, will it call me?
---
Right now your pipeline is set up like this:
	1.	WIX form → sends to your ingest endpoint (/api/webhooks/wix) ✅
→ This stores/normalizes the lead in memory/Redis.
	2.	ElevenLabs initiation webhook → points to your fetch endpoint (/api/elevenlabs/init) ✅
→ When a Twilio call starts, Penny can pull the normalized lead data.

⚠️ What’s missing: something to actually initiate the outbound call.
	•	Submitting a WIX form only caches the lead.
	•	For a call to fire, you need either:
	•	Penny configured in ElevenLabs to auto-call the lead_phone after ingest, or
	•	Your ingest endpoint triggers the ElevenLabs/Twilio outbound call API once a new lead is stored.

👉 Next step: add the outbound call trigger. That means wiring your ingest (/api/webhooks/wix) to POST to ElevenLabs/Twilio with the new lead’s number, so Penny starts dialing.

Do you want me to sketch that outbound trigger code so your ingest automatically tells ElevenLabs/Twilio to call the new lead?
