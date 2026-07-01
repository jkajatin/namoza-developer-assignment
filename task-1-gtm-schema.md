# Task 01 — GTM Event Schema
### OrthoNow · Developer Assignment · Namoza

---

## 1. Event Schema

| # | Event Name | Trigger Type (GTM) | Key Parameters | GA4 Report / Audience Fed |
|---|---|---|---|---|
| 1 | `booking_step_complete` | Custom Event — dataLayer push fired by the front-end on each step transition (GTM cannot detect step changes in a multi-step form natively) | `step_number`, `step_name`, `clinic_location`, `specialty` (step 1 only), `form_name`, `lead_source` | GA4 Funnel Exploration (booking funnel); feeds "Booking Started" audience (`step_number = 1`) |
| 2 | `booking_confirmed` *(derived — see §3)* | GA4-derived event, condition `booking_step_complete` + `step_number = 3` | `clinic_location`, `specialty`, `preferred_date`, `form_name`, `lead_source` | Conversions report; recommended Google Ads conversion import; feeds "Converted Leads" audience |
| 3 | `call_now_click` | Click — Just Links, triggered on `tel:` href pattern, filtered to `.call-now-btn` class to exclude unrelated tel links | `page_location`, `page_type` (home / clinic / landing), `clinic_name` (if on a clinic page), `lead_source` | Engagement report, segmented by page type; feeds "High Intent — Call" audience |
| 4 | `whatsapp_widget_click` | Click — Just Links, triggered on href containing `wa.me` | `page_location`, `page_type`, `click_position` (floating widget vs inline), `lead_source` | Engagement report; feeds "WhatsApp Engaged" audience for retargeting |
| 5 | `patient_guide_lead_submit` | Custom Event — dataLayer push on gate-form submit, before the PDF unlocks | `page_location`, `guide_name`, `form_name`, `lead_source` | Lead gen report; feeds "Guide Lead" audience (mid-funnel nurture) |
| 6 | `patient_guide_download` | GTM's built-in File Download trigger, filtered to `.pdf` | `file_name`, `link_url`, `page_location` | Content engagement report; cross-referenced against #5 to measure gate-to-download completion |
| 7 | `clinic_page_view` | Page View (or History Change if the site is a SPA) filtered by URL path `/clinics/*`; fires once per clinic page load | `clinic_name`, `city`, `page_location` | Pages and Screens report broken out per clinic; feeds location-based remarketing audiences (9 clinics) |
| 8 | `blog_scroll_depth` | GTM Scroll Depth Trigger (25%, 50%, 75%, 90%) | `scroll_percentage`, `article_title`, `article_category` | Engagement report (scroll depth breakdown); feeds "Engaged Reader" audience for content remarketing |

**Note on row 1:** all three booking steps share a single event name, differentiated by `step_number`. This is what allows the entire journey to be built as one GA4 Funnel Exploration rather than three separately-tracked events. See §2 for the implementation detail and §3 for how this decision is reconciled with the Google Ads import requirement.

---

## 2. Tracking Funnel Drop-off in the 3-Step Booking Form

### Implementation approach

GTM cannot detect transitions between steps in a JavaScript multi-step form on its own. The front-end developer must push a custom event to `window.dataLayer` whenever a user successfully completes each step.

GTM then listens for these custom events using a Custom Event Trigger named `booking_step_complete`. Each push contains the current step number and metadata about that step. GA4 receives a single event (`booking_step_complete`) with different parameter values, making it easy to build one funnel instead of stitching together multiple event names.

### Step 1 — Clinic & Specialty Selected

```json
{
  "event": "booking_step_complete",
  "step_number": 1,
  "step_name": "location_specialty_selected",
  "clinic_location": "Whitefield",
  "specialty": "Knee Replacement",
  "form_name": "consultation_booking",
  "lead_source": "Google Ads"
}
```

### Step 2 — Patient Details Submitted

```json
{
  "event": "booking_step_complete",
  "step_number": 2,
  "step_name": "patient_details_entered",
  "preferred_date": "2026-07-15",
  "form_name": "consultation_booking",
  "lead_source": "Google Ads"
}
```

`name` and `phone` are intentionally excluded — they are personally identifiable information (PII) and should not be sent to GA4/GTM.

### Step 3 — Booking Confirmed

```json
{
  "event": "booking_step_complete",
  "step_number": 3,
  "step_name": "booking_confirmed",
  "clinic_location": "Whitefield",
  "specialty": "Knee Replacement",
  "preferred_date": "2026-07-15",
  "form_name": "consultation_booking",
  "lead_source": "Google Ads"
}
```

### GTM Configuration

For all three steps:

- **Trigger Type:** Custom Event
- **Event Name:** `booking_step_complete`
- **Variables:** Data Layer Variables created for `step_number`, `step_name`, `clinic_location`, `specialty`, `preferred_date`, `form_name`, `lead_source`

These variables are then passed to the GA4 Event Tag.

### GA4 Funnel Exploration

Create an Open Funnel in GA4 using the same event (`booking_step_complete`), filtering each step by the `step_number` parameter:

| Funnel Step | Event | Filter |
|---|---|---|
| Step 1 | `booking_step_complete` | `step_number = 1` |
| Step 2 | `booking_step_complete` | `step_number = 2` |
| Step 3 | `booking_step_complete` | `step_number = 3` |

This allows GA4 to report: users reaching Step 1, drop-off before Step 2, drop-off before Step 3, and final completion rate. Because every step shares the same event name — differentiated only by `step_number` — the funnel remains easy to maintain and extend if a fourth step is ever added.

---

## 3. Google Ads Conversion Import

**Recommended conversion: `booking_confirmed`**

The front-end pushes a single `booking_step_complete` event throughout the booking journey, differentiated by the `step_number` parameter. This keeps the data layer consistent and allows the entire booking journey to be analysed as one GA4 Funnel Exploration.

However, Google Ads imports GA4 **events**, not parameter-filtered instances of an event. Marking `booking_step_complete` itself as a Key Event would count every step — including Step 1 — as a conversion, which defeats the purpose of optimising toward completed bookings only.

To resolve this without compromising the dataLayer design, I would create a **derived GA4 event**:

- **Source event:** `booking_step_complete`
- **Condition:** `step_number = 3`
- **New event name:** `booking_confirmed`

This derived event is then marked as the Key Event (Conversion) in GA4 and imported into Google Ads.

**Why this event, and why derived rather than fired separately from the front-end:**

Earlier events such as Step 1 or Step 2 indicate intent but produce false positives if users abandon the process — importing those would optimise campaigns toward users who only *begin* booking, not complete it. The final confirmation state is the highest-quality conversion signal for bidding and audience optimisation.

Deriving `booking_confirmed` inside GA4 — rather than having the front-end fire a second, duplicate event at Step 3 — keeps the dataLayer to a single clean event for the entire booking flow, avoids redundant pushes at the same user action, and puts the conversion-definition logic where it belongs: in GA4's configuration layer, not the front-end code. If a second conversion definition is ever needed later (e.g. "reached Step 2"), it can be derived the same way without touching the dataLayer again.
