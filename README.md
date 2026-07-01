# Namoza Developer Assignment – Jatin Agrawal

This repository contains my submission for the Namoza Developer (Client Web + Martech) assignment.

## Repository Structure

```
.
├── task-1-gtm-schema.md
├── task-2/
│   ├── index.html
│   └── pagespeed-mobile-90+.png
├── task-3-integration.md
└── README.md
```

## Task 1 – GTM Event Schema

Designed a complete GA4/GTM event architecture covering:

- Multi-step booking funnel
- Call tracking
- WhatsApp interactions
- PDF lead generation
- Clinic page tracking
- Blog engagement

Highlights:

- Single `booking_step_complete` event parameterized by `step_number`
- Front-end driven `dataLayer.push()` implementation
- GA4 Funnel Exploration configuration
- Derived `booking_confirmed` event for Google Ads conversion import

---

## Task 2 – Landing Page

Built a self-contained HTML/CSS/JavaScript landing page.

Features:

- Mobile-first layout
- Minimal 2-field lead form
- No external assets or frameworks
- Client-side validation
- Thank-you state without reload
- `window.dataLayer.push()` on successful submission
- No PII sent to GA4/GTM
- Optimized for 90+ Mobile PageSpeed

---

## Task 3 – CRM Integration

Designed an end-to-end integration using a direct API architecture.

Flow:

Landing Page
→ Serverless Endpoint
→ HubSpot CRM
→ Karix WhatsApp API
→ Google Ads Conversion

Highlights:

- Search-before-create using HubSpot CRM Search API
- Phone-number deduplication via custom unique property
- Queue/retry mechanism
- SLA monitoring for WhatsApp delivery
- Conversion fired only after successful backend processing

---

## Design Philosophy

Although each task was completed independently, I kept the architecture consistent across all three.

- Task 2 avoids sending PII into the data layer.
- Task 1 uses a single clean analytics event and derives conversions inside GA4.
- Task 3 only fires the Google Ads conversion after backend success, ensuring conversion data represents successful lead processing rather than optimistic form submissions.

The common principle throughout is maintaining accurate analytics and reliable marketing signals while keeping the implementation simple and maintainable.
