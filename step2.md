What “Fetch initiation client data from webhook” actually does

When a call begins (via Twilio → ElevenLabs), ElevenLabs makes an HTTP request to your URL. Whatever JSON you return is merged into the conversation’s client variables. If your keys match your ElevenLabs variable/Data Collection identifiers, those values are immediately available for:

prompt variables like {{first_name}}, {{lead_phone}}, etc.

your Data Collection items (so they’re saved with the transcript)

So the task is: build a small webhook that returns normalized fields from your WIX payload.

The reliable pattern (two tiny endpoints)
1) Ingest endpoint (WIX → you)

Point your WIX webhook (currently hitting webhook.site) to your endpoint, store the raw payload, and normalize it.
Key the record by something stable (submissionId, phone E.164, or both).

Normalization goals from your sample:

first_name: field:first_name_b9ed or from contact.name.first

last_name: field:last_name_072b or from contact.name.last

lead_phone (E.164): +12814562323 (use contact.phones[0].e164Phone if present)

address_line1: contact.address.addressLine (or first line of formattedAddress)

city: contact.address.city

state: contact.address.subdivisions[0].code → “TX”

zip: contact.address.postalCode

notes: “i think hes a pee-doh-file” (sanitize)

request_type: form name or project field ("Tile Roofing" or "There is a martian on my roof")

location: "Dallas-Fort Worth"

source_site: "wix_form"

2) Fetch-for-ElevenLabs endpoint (ElevenLabs → you)

Enable the switch in Security → “Fetch initiation client data from webhook”, and set this endpoint URL.
When ElevenLabs calls it (at call start), look up the lead (usually by the Twilio caller/callee E.164 present in the request; log the request the first time so you can see exactly which field they provide), and return a flat JSON map with the identifiers Penny expects.
