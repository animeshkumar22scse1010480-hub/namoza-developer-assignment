# Task 01 — OrthoNow GTM Event Schema

## Q1. Complete GTM Event Schema

| Event Name | GTM Trigger Type | Key Parameters (min. 3) | GA4 Report / Audience / Purpose | Conversion? |
|---|---|---|---|---|
| booking_started | Custom Event (`booking_started`) via dataLayer | clinic_location, specialty, page_url, device_type | Funnel Exploration – Booking Started | No |
| booking_step_complete | Custom Event (`booking_step_complete`) via dataLayer | step_number, step_name, clinic_location, specialty, page_url | Funnel Exploration – Step Completion | No |
| booking_confirmed | Custom Event (`booking_confirmed`) via dataLayer | appointment_id, clinic_location, specialty, preferred_date, booking_source | Conversions → Appointment Bookings | ✅ Yes |
| booking_abandoned | Timer Trigger (30s inactivity) + Custom Event | last_completed_step, clinic_location, specialty, page_url | Funnel Drop-off Analysis | No |
| booking_error | Custom Event (`booking_error`) | error_message, step_number, clinic_location, page_url | Debug & Error Monitoring | No |
| call_now_click | Click – Just Links (Click URL starts with `tel:`) | clinic_location, phone_number, page_url, device_type, button_position | High Intent Users Audience | No |
| whatsapp_chat_click | Click – Just Links (Click URL contains `wa.me`) | clinic_location, page_url, device_type, button_position | WhatsApp Engagement Audience | No |
| patient_guide_form_started | Form Interaction (First Input) | page_url, clinic_location, device_type | Lead Funnel Analysis | No |
| patient_guide_download | PDF Download Trigger (Click URL ends with `.pdf`) | guide_name, clinic_location, phone_number, page_url | Lead Generation Conversion | ✅ Yes |
| clinic_page_view | Page View (Page Path contains `/clinics/`) | clinic_location, city, page_title, referrer | Location Interest Audience | No |
| blog_scroll_25 | Scroll Depth Trigger (25%) | article_title, article_category, scroll_percent | Content Engagement | No |
| blog_scroll_50 | Scroll Depth Trigger (50%) | article_title, article_category, scroll_percent | Blog Engagement Report | No |
| blog_scroll_75 | Scroll Depth Trigger (75%) | article_title, article_category, scroll_percent | Engaged Readers Audience | No |
| blog_scroll_90 | Scroll Depth Trigger (90%) | article_title, article_category, scroll_percent | Highly Engaged Audience | No |
| cta_click | Click Trigger (CSS Selector / Button Class) | button_text, page_url, section_name, device_type | CTA Performance Analysis | No |
| outbound_link_click | Click – Just Links (URL does not contain orthonow domain) | destination_url, link_text, page_url | Outbound Link Report | No |
| form_validation_error | Custom Event (`form_validation_error`) | field_name, error_type, form_name, page_url | Form Optimization Report | No |
| session_engaged | GA4 Enhanced Measurement | engagement_time_msec, page_views, session_id | Engagement Report | No |
| first_visit | GA4 Automatic Event | source, medium, landing_page | New Users Audience | No |
| returning_user | GA4 Automatic Event | source, medium, previous_sessions | Returning Users Audience | No |

---

## Q2. 3-Step Booking Form — Funnel Drop-off Tracking

The appointment booking process consists of three steps:

1. Select Clinic Location + Specialty
2. Enter Patient Details (Name, Phone Number, Preferred Date)
3. Confirm Booking

Since this is a multi-step form, **GTM cannot automatically detect when a user completes each step**. GTM only listens for events — it does not know a "step" happened unless the front-end explicitly tells it. Therefore, the front-end developer must implement `window.dataLayer.push()` whenever a user successfully completes a step. GTM listens for these custom events, forwards them to GA4, and GA4 builds the Funnel Exploration report from them.

### Step 1 – Select Clinic Location & Specialty

**Front-end action:** When the user selects a clinic location and specialty and clicks "Next", after validation passes, the front-end fires:

```json
{
  "event": "booking_step_complete",
  "step_number": 1,
  "step_name": "location_specialty_selected",
  "clinic_location": "{{clinic name}}",
  "specialty": "{{specialty selected}}",
  "page_url": "{{page path}}",
  "device_type": "{{device category}}"
}
```

**GTM Trigger:** Custom Event Trigger — Event name equals `booking_step_complete`, Condition: `step_number` equals `1`
**GA4 Event:** Send Event (GA4 Event Tag), event name `booking_step_complete`
**Parameters mapped:** step_number, step_name, clinic_location, specialty, page_url, device_type

### Step 2 – Patient Details

**Front-end action:** After the patient enters Name, Phone Number, and Preferred Appointment Date and clicks "Next":

```json
{
  "event": "booking_step_complete",
  "step_number": 2,
  "step_name": "patient_details_entered",
  "clinic_location": "{{clinic name}}",
  "specialty": "{{specialty selected}}",
  "preferred_date": "{{preferred appointment date}}",
  "page_url": "{{page path}}"
}
```

**GTM Trigger:** Custom Event Trigger — Event name equals `booking_step_complete`, Condition: `step_number` equals `2`
**GA4 Event Parameters:** step_number, step_name, clinic_location, specialty, preferred_date, page_url

> Note: Name and Phone Number are **never** pushed into the dataLayer — only non-PII booking metadata is sent to GTM/GA4, to avoid sending personal data into Google's systems.

### Step 3 – Booking Confirmation

**Front-end action:** When the booking is successfully submitted and the confirmation page displays:

```json
{
  "event": "booking_confirmed",
  "appointment_id": "{{appointment id}}",
  "clinic_location": "{{clinic name}}",
  "specialty": "{{specialty selected}}",
  "preferred_date": "{{preferred appointment date}}",
  "booking_source": "{{utm source / campaign}}"
}
```

**GTM Trigger:** Custom Event Trigger — Event name equals `booking_confirmed`
**GA4 Event Parameters:** appointment_id, clinic_location, specialty, preferred_date, booking_source
This event is marked as a **GA4 Conversion** and imported into Google Ads.

### GTM Implementation Flow

```
User completes a step
        │
        ▼
Front-end validates inputs
        │
        ▼
window.dataLayer.push()
        │
        ▼
GTM Custom Event Trigger fires
        │
        ▼
GA4 Event Tag fires
        │
        ▼
GA4 receives event
        │
        ▼
Funnel Exploration updates
```

The same process repeats for Step 2 and Step 3, each with its own `step_number`.

### GA4 Funnel Exploration Setup

Navigate: **GA4 → Explore → Funnel Exploration**

Build the funnel using these steps:

| Funnel Step | Event Name | Condition |
|---|---|---|
| Step 1 | booking_step_complete | step_number = 1 |
| Step 2 | booking_step_complete | step_number = 2 |
| Step 3 | booking_confirmed | event_name = booking_confirmed |

**Example drop-off output:**

| Funnel Step | Users | Drop-off |
|---|---|---|
| Step 1 Completed | 1,000 | — |
| Step 2 Completed | 720 | 28% |
| Booking Confirmed | 510 | 29% |

From this report: 28% of users abandon after selecting a clinic and specialty; 29% abandon after entering patient details; overall booking conversion is 51% (510 of 1,000 users). This tells the team exactly where friction exists in the form so it can be optimized.

---

## Q3. Which Conversion to Import into Google Ads

**Chosen conversion: `booking_confirmed`**

**Reason:** `booking_confirmed` represents a completed appointment booking — OrthoNow's actual business goal, not just an engagement signal. It's a high-intent, bottom-funnel event and gives Google Ads Smart Bidding (Maximize Conversions / Target CPA) the cleanest signal to optimize toward.

Events like `call_now_click`, `whatsapp_chat_click`, and `patient_guide_download` are valuable **micro-conversions** for audience building and remarketing — but none of them confirm an appointment was actually booked, so importing those as the primary Ads conversion would mislead the bidding algorithm into optimizing for the wrong outcome.

| Conversion Event | Import into Google Ads? | Reason |
|---|---|---|
| booking_confirmed | ✅ Yes | Represents a completed appointment booking, directly aligned with OrthoNow's core KPI |
| call_now_click | ❌ No | Indicates interest, not a confirmed booking |
| whatsapp_chat_click | ❌ No | Measures engagement only |
| patient_guide_download | ❌ No | Useful for lead generation and remarketing, not a confirmed appointment |

---

## Developer Briefing Note

**Who writes the `dataLayer.push()`?**
The front-end developer, not GTM. GTM cannot detect progress through a multi-step form on its own — it only reacts to events that already exist in the dataLayer. Someone has to put them there.

**Brief for implementing Step 2 specifically:**
- Fire `dataLayer.push()` only after the user successfully completes Step 2 (Name, Phone Number, Preferred Date) and clicks "Next."
- Validate all required fields *before* firing the event — don't fire on an invalid submit attempt.
- The event should fire exactly once per successful step completion — never on page load, field focus, or every keystroke.
- GTM listens for the custom event `booking_step_complete` with `step_number = 2` and forwards it to GA4 for funnel analysis and drop-off tracking.

This ensures GA4 Funnel Exploration reports accurate, reliable step-by-step progress through the booking flow.

---

## Extra: GA4 Event Naming Strategy

- Use `snake_case` for all event names (e.g., `call_now_click`).
- Use action-oriented names that clearly describe user behavior.
- Prefer GA4 recommended events where one exists (e.g., `generate_lead`).
- Create custom events only when no suitable GA4 recommended event exists (e.g., `whatsapp_chat_click`).

**Recommended vs. Custom Event Mapping:**

| Event | GA4 Recommended Event | Custom Event | Notes |
|---|---|---|---|
| booking_confirmed | purchase | No | Represents a completed appointment booking |
| patient_guide_download | generate_lead | No | Tracks successful lead generation |
| call_now_click | — | Yes | High-intent micro-conversion |
| whatsapp_chat_click | — | Yes | Tracks WhatsApp engagement |
| blog_scroll_depth | — | Yes | Measures content engagement |

## Extra: Step-by-Step GTM Implementation Workflow

1. Define business goals and map them to GA4 recommended or custom events.
2. Implement `dataLayer.push()` for custom booking events through the front-end.
3. Create GA4 Event Tags in GTM for each event.
4. Configure appropriate GTM triggers (clicks, form submissions, scroll depth, custom events).
5. Test all events using GTM Preview Mode.
6. Verify event collection in GA4 Realtime and DebugView.
7. Mark key events (`booking_confirmed`, `patient_guide_download`) as Conversions.
8. Link GA4 with Google Ads and import the primary conversion for campaign optimization.

## Extra: Audience Use Cases

- **High-Intent Audience:** Users who clicked Call Now or WhatsApp Chat.
- **Lead Nurturing Audience:** Users who downloaded the Patient Guide but did not book an appointment.
- **Location-Specific Audience:** Users who viewed individual clinic location pages.
- **Content Engagement Audience:** Users who scrolled 75%+ on blog articles.
- **Booking Abandoners:** Users who completed Step 1 or Step 2 but did not confirm their booking (for remarketing campaigns).
