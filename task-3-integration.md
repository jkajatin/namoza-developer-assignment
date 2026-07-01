# Task 03 — Integration Design
### OrthoNow · Developer Assignment · Namoza

## Architecture

The landing page form posts to a lightweight serverless function (a Vercel/Lambda endpoint) acting as the integration layer between the front-end, HubSpot, and Karix. I chose a **direct API call** over Zapier/Make/native HubSpot embed because the dedup logic below needs a conditional lookup-before-write that no-code tools handle poorly, and the 2-minute SLA leaves no room for their added polling latency.

Flow, in order:

1. Form submits `{ name, phone, clinic_preference }` to the endpoint.
2. The endpoint queries HubSpot's CRM Search API by a **custom unique property, `phone_number`** — not email. **This is deliberate: HubSpot's default dedup key is email, and this form never collects one.** Left unaddressed, every repeat submission creates a duplicate contact. `phone_number` must be created in HubSpot as a unique-value custom property and searched before deciding create vs. update.
3. Match found → `PATCH` the contact: update Lead Status, append a new timeline note (not overwrite Name — two people can share a phone, e.g. a home number). No match → `POST` a new contact with Name, Phone, Clinic Preference, Source = "Google Ads – Consultation Landing Page", Lead Status = "New Enquiry".
4. In parallel, the endpoint calls Karix's WhatsApp API with the confirmation template.
5. Only after the endpoint returns success does the front-end fire the `booking_confirmed` Google Ads conversion — firing it optimistically on submit, before HubSpot/Karix confirm, would let failed leads pollute the conversion signal.

**Same phone, different names:** the search-by-`phone_number` step means the second submission updates the existing contact, not a new one. Name isn't silently overwritten — the update logs as a new timeline note and resets Lead Status to "New Enquiry," so the coordinator sees both attempts and verifies identity on the call.

## Biggest failure point

The integration endpoint — a single point of failure between two independent third parties. If it errors, HubSpot and Karix fail together, silently. Fallback: write every payload to a durable queue *before* calling either API; a retry worker reprocesses failures asynchronously. The user still sees the thank-you state immediately — failure handling never blocks the lead.

## SLA monitoring

What breaks the 2-minute WhatsApp SLA: Karix rate limits, template approval lag, or queue backlog during traffic spikes. I'd log submit timestamp vs. Karix's delivery webhook timestamp and alert (Slack/PagerDuty) whenever that gap exceeds 90 seconds — ahead of the SLA breach, not after.
