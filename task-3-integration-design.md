# Task 03 — Integration Design

## How I would architect this integration end-to-end

The form submits to a lightweight custom middleware endpoint (a serverless function, e.g. on Vercel or AWS Lambda) rather than a native HubSpot embed or a no-code tool like Zapier or Make. I chose a **direct API call to custom middleware** because this flow is SLA-critical (the WhatsApp message must fire within 2 minutes) and requires custom deduplication logic that HubSpot does not handle out of the box. No-code automation tools add webhook processing latency and offer far less control over retries, error handling, and timing than an endpoint I own and control directly.

**The flow runs in this order:**

1. The form submits to the middleware endpoint — not directly to HubSpot.
2. The middleware searches HubSpot Contacts **by phone number**, using the CRM Search API. This is deliberate: HubSpot's default deduplication is based on email, and this form never collects an email address, so relying on HubSpot's default behavior would silently create duplicate contacts for every returning patient.
3. If a matching contact is found by phone, it is updated. If not, a new contact is created with Name, Phone, Clinic Preference, Source ("Google Ads – Consultation Landing Page"), and Lead Status ("New Enquiry").
4. The middleware then calls Namoza's WhatsApp Business API (via Karix) to send the patient a confirmation message.
5. The Google Ads conversion event ("consultation_form_submitted") fires client-side via gtag immediately on form submit, independent of the backend steps, so ad platform optimization is never delayed by CRM or WhatsApp processing time.

## The single biggest failure point, and how I would build a fallback

The middleware endpoint itself is the single point of failure — if it goes down, all three downstream actions (HubSpot update, WhatsApp send, and conversion tracking) fail together.

To fix this, I would decouple the steps: the endpoint's only job is to validate the incoming lead and place it on a queue (e.g., AWS SQS). Separate worker processes then handle the HubSpot upsert and the WhatsApp send independently, each with its own retry logic and a dead-letter queue for failures. This way, a failure in one leg — for example, HubSpot being temporarily unreachable — does not block or delay the WhatsApp confirmation, and vice versa.

## What could break the 2-minute WhatsApp SLA, and how I would monitor it

The main risks are: Karix API rate limits or downtime, WhatsApp template throttling on non-session messages, cold starts on the serverless middleware, or queue backlog during high-traffic periods from paid campaigns.

To monitor this, I would log a `created_at` timestamp when the lead is captured and a `whatsapp_sent_at` timestamp when the message is confirmed sent, and set up an alert if that gap exceeds 90 seconds — giving a buffer before the 2-minute SLA is actually breached. I would also track Karix's API error rate on a dashboard so failures are visible immediately rather than discovered from a patient complaint.

## On the phone number deduplication trap

If two patients submit the same phone number with different names, the system does **not** silently overwrite the existing contact's identity. Instead, it updates the contact but logs the previous name as a note on the HubSpot timeline, flagging the mismatch for the clinic coordinator to verify manually on the confirmation call — since two people legitimately sharing a phone number (a common scenario in Indian households) is a real possibility that shouldn't be resolved automatically.
