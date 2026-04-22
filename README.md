# Supermetrics Data Activation - Web Tag

This Google Tag Manager (GTM) template provides a seamless, script-free way to integrate your website with the Supermetrics Data Activation Customer Data Platform (CDP). 

Instead of loading a heavy custom JavaScript tracking snippet on your website, this template natively manages user identity, engagement tracking, e-commerce events, and real-time audience matching directly through GTM.

## 📋 Prerequisites
Before configuring your tags, you will need:
* **Your Site ID:** A unique number identifying your CDP instance (found in your Data Activation admin panel).
* **A Tracking Identifier (UUID):** A stable identifier for the user. We highly recommend using this template to generate a first-party `_sm_da_uuid` cookie (see *Tag Types* below).

---

## 🏷️ Choosing Your Tag Type

This template is modular. You will create multiple tags in your GTM container using this single template by selecting different "Tag Types" from the dropdown.

### 1. Generate `_sm_da_uuid` cookie (Identity Management)
**What it does:** This is the foundation of your tracking. It generates a unique, anonymous identifier (UUID) for new visitors and manages the browser cookie to ensure users are tracked consistently across sessions.
* **Best Practice:** Create one tag with this type and set it to fire on **All Pages** as early as possible (e.g., Initialization or Page View).
* **Server GTM:** To protect your tracking from Safari's Intelligent Tracking Prevention (ITP) which deletes cookies after 7 days, expand the **Server GTM configuration** section and enter your **Server container URL**. This routes the cookie generation through your own Server GTM domain, granting it a stable, long-term lifespan. You will require to import our [`Supermetrics Data Activation - Server Client.tpl` 🔗](https://github.com/orgs/Supermetrics-Professional-Services/repositories) into your sGTM container to implement the server-set cookie creation. 

### 2. Engagement
**What it does:** Tracks real-time, timestamped user interactions on your website. Engagements are used to trigger journey orchestrations or build behavioral audiences.
* **Examples:** `page_view`, `button_click`, `form_submission`, `video_play`.
* **Configuration:** Provide an Engagement Name and map relevant properties (e.g., Property Name: `page_url`, Property Value: `{{Page URL}}`).

### 3. Fact
**What it does:** Stores long-term properties that describe a user across multiple sessions, rather than a single point-in-time action. 
* **Examples:** `loyalty_tier`, `age`, `last_category_viewed`.
* **Time To Live (TTL):** Facts require a TTL (in seconds). This ensures your CDP doesn't store stale data. For example, a user's `last_category_viewed` might only be relevant for 30 days (2,592,000 seconds), after which the CDP automatically clears it.

### 4. Mapping (Identity Resolution)
**What it does:** Connects additional identifiers (like a CRM ID, hashed email, or Facebook cookie) to the user's anonymous UUID profile. 
* **Partner Type:** The assigned slot ID for the identifier (e.g., `7001` for an email hash, `7002` for a phone hash).
* **Merge Profiles:** If checked, the CDP will look for an existing profile associated with this new ID and seamlessly merge it with the user's current anonymous web profile, unifying their journey. Only use this for primary identifiers (like a hashed email upon login/signup).

### 5. Data Layer E-commerce events
**What it does:** Automatically captures e-commerce data and routes it to the CDP without requiring you to manually map individual product properties.
* **Requirement:** Your website must push e-commerce data to the dataLayer following the standard [**Google Analytics 4 (GA4) Enhanced Ecommerce** 🔗](https://developers.google.com/analytics/devguides/collection/ga4/reference/events?client_type=gtag#online_sales) schema.
* **Supported Events:** `view_item`, `add_to_wishlist`, `add_to_cart`, `remove_from_cart`, `view_cart`, `begin_checkout`, `purchase`.

### 6. Audience Match - Custom event
**What it does:** Queries the CDP in real-time to check if the current user belongs to specific audiences (segments), allowing you to personalize the website or trigger specific third-party marketing tags.
* **Configuration:** Input a comma-separated list of Audience API IDs (e.g., `1234_1, 1234_2`).
* **The Output:** The tag always pushes a custom event named `smda_audience_match` to the GTM dataLayer, regardless of whether any audience matched. It includes two parameters:
  * `matched_audience_ids` — an array of the IDs of all matched audiences (e.g. `['1234_1', '1234_3']`), or `null` if none matched.
  * `matched_audiences_data` — an object keyed by matched audience ID, each containing the segment payload returned by the CDP (e.g. `{'1234_1': {'lastProduct': 'snowboarding'}}`), or `null` if none matched.
* **Use Case:** Create downstream Custom HTML tags that fire on the `smda_audience_match` event. Use `matched_audience_ids` to identify which audiences matched and `matched_audiences_data` to access segment-specific properties for personalisation or targeting.

#### Setting up downstream activation tags

Before creating any activation tag, you need **two GTM variables** to expose the event parameters:

1. In GTM, go to **Variables → New → Data Layer Variable**.
2. Set **Data Layer Variable Name** to `matched_audience_ids`. Name the variable `dl - matched_audience_ids` and save.
3. Repeat for a second variable: set **Data Layer Variable Name** to `matched_audiences_data`. Name it `dl - matched_audiences_data` and save.

Then, for each activation tag below:

* **Tag type:** Custom HTML
* **Trigger:** Custom Event — Event Name: `smda_audience_match`

---

#### Example A — Meta (Facebook) Pixel: fire a custom event per matched audience

This tag initialises the Meta pixel (if not already present on the page) and fires one `trackCustom` event per matched audience ID. The audience ID is passed as a parameter so you can use it as a custom signal in Facebook Ads Manager.

```html
<script>
(function() {
  // Replace with your Facebook Pixel ID
  var fb_id = '1111111111111111';

  // Matched audiences from the smda_audience_match dataLayer event
  var matched_audiences = {{dl - matched_audience_ids}} || [];

  // Load Facebook pixel stub if not already loaded
  if (!window.fbq) {
    (function(f, b, e, v, n, t, s) {
      if (f.fbq) return;
      n = f.fbq = function() {
        if (!n.callMethod) {
          n.queue.push(arguments);
        } else {
          n.callMethod.apply(n, arguments);
        }
      };
      if (!f._fbq) f._fbq = n;
      n.push = n;
      n.loaded = !0;
      n.version = '2.0';
      n.queue = [];
      t = b.createElement(e);
      t.async = !0;
      t.src = v;
      s = b.getElementsByTagName(e)[0];
      s.parentNode.insertBefore(t, s);
    })(window, document, 'script', 'https://connect.facebook.net/en_US/fbevents.js');
  }

  fbq('init', fb_id);

  // Fire a custom event for each matched audience ID
  for (var i = 0; i < matched_audiences.length; i++) {
    fbq('trackCustom', 'SM_DA_audience', { Audience: matched_audiences[i] });
  }

})();
</script>
```

> **Note:** If you already have a standard Meta base pixel tag firing on all pages, you do not need to include the pixel stub above — just keep the `fbq('init', ...)` and `fbq('trackCustom', ...)` calls.

---

#### Example B — Google Ads: fire a remarketing conversion per matched audience

This tag maps each Data Activation Audience ID to a specific Google Ads conversion label and fires a remarketing-only conversion event for each match. A 4-hour localStorage throttle prevents the same conversion from firing repeatedly on subsequent page views within the same session.

**Prerequisites:** A Google Ads tag or Google Tag (`gtag.js`) must already be loaded on the page, defining `window.gtag`.

```html
<script>
(function() {
  // Map your Data Activation Audience IDs to their Google Ads conversion labels
  var mapping = {
    '1090_1': 'AW-111111111/AxxxxxxxxxADEJ6r38MD',
    '1090_2': 'AW-111111111/BxxxxxxxxxADEJ6r38MD',
    '1090_3': 'AW-111111111/CxxxxxxxxxADEJ6r38MD',
    '1090_4': 'AW-111111111/DxxxxxxxxxADEJ6r38MD'
  };

  // Guard: gtag must be available (loaded by your Google Ads / Google Tag)
  if (typeof window.gtag !== 'function') return;

  // Matched audiences from the smda_audience_match dataLayer event
  var matched_audiences = {{dl - matched_audience_ids}} || [];

  // Throttle: avoid re-firing the same conversion more than once every 4 hours
  var _sm_da_sent = {};
  try {
    _sm_da_sent = JSON.parse(localStorage.getItem('_sm_da_sent')) || {};
  } catch(e) {}

  var now = +new Date();
  var fourHours = 14400000;

  for (var i = 0; i < matched_audiences.length; i++) {
    var cur_audience_id = matched_audiences[i];
    var conversionLabel = mapping[cur_audience_id];

    // Skip if this audience has no Google Ads mapping
    if (!conversionLabel) continue;

    // Skip if this conversion was already fired within the last 4 hours
    if (_sm_da_sent[cur_audience_id] && (_sm_da_sent[cur_audience_id] > now - fourHours)) continue;

    // Fire the Google Ads remarketing conversion
    gtag('event', 'conversion', {
      send_to: conversionLabel,
      aw_remarketing_only: true
    });

    _sm_da_sent[cur_audience_id] = now;
  }

  try {
    localStorage.setItem('_sm_da_sent', JSON.stringify(_sm_da_sent));
  } catch(e) {}

})();
</script>
```

> **How to find your conversion label:** In Google Ads, go to **Goals → Conversions → New conversion action → Website**. The `send_to` value is shown in the tag snippet as `AW-XXXXXXXXX/YYYYYYYYYYY`. Use `aw_remarketing_only: true` to signal that this conversion is for audience targeting only, not bid optimisation.

### 7. Orchestration Match - Custom event
**What it does:** Queries the CDP in real-time to check whether the current user is part of a specific Orchestration Journey or Audience, and evaluates their position against a configured Step condition. This allows you to trigger personalisation or third-party tags based on where a user is (or is not) within a CDP-managed journey.

* **Configuration:**
  * **Journey or Audience ID** — the UUID of the orchestration journey or audience to query.
  * **Step evaluation** — the condition to evaluate:
    * **In ANY step** — matches if the user is found anywhere in the journey, regardless of their current step.
    * **IN a specific step** — matches if the user's current step equals the configured Step ID.
    * **NOT in a specific step** — matches if the user's current step differs from the configured Step ID.
  * **Step ID** — the UUID of the step to evaluate against. Required for `IN a specific step` and `NOT in a specific step`.

* **The Output:** The tag always pushes a custom event named `smda_orchestration_match` to the GTM dataLayer, regardless of whether the condition matched. It includes the following parameters:
  * `orchestration_match` — `true` if the condition matched, `false` if not. Always present.
  * `orchestration_id` — the configured Journey/Audience ID. Always present.
  * `matched_orchestration_condition` — the evaluation mode configured in the tag (`any_step`, `in_step`, `not_in_step`). Always present.
  * `step_id` — the configured Step ID. Present only when the condition is `in_step` or `not_in_step`.
  * `orchestration_data` — the full journey payload returned by the CDP if matched (including `currentStepId` and any segment properties), or `null` if not matched.

* **Use Case:** Build GTM triggers scoped to a specific journey and step by combining `orchestration_id`, `step_id`, and `orchestration_match equals true`. Use `orchestration_data` in downstream tags to access any segment-level properties returned by the CDP.

#### Setting up downstream activation tags

Create **GTM Data Layer Variables** for each parameter you want to use in downstream tags or trigger conditions:

| Variable name | Data Layer Variable Name |
|---|---|
| `dl - orchestration_match` | `orchestration_match` |
| `dl - orchestration_id` | `orchestration_id` |
| `dl - matched_orchestration_condition` | `matched_orchestration_condition` |
| `dl - step_id` | `step_id` |
| `dl - orchestration_data` | `orchestration_data` |

**Recommended trigger setup for a downstream tag:**
* **Tag type:** Custom HTML
* **Trigger:** Custom Event — Event Name: `smda_orchestration_match`
* **Additional condition:** `dl - orchestration_match` **equals** `true`
* Optionally, further scope the trigger by adding `dl - orchestration_id` **equals** `<your-journey-uuid>` and `dl - step_id` **equals** `<your-step-uuid>`.

---

## 🛡️ Server GTM Configuration (Highly Recommended)

Modern browsers strictly limit the lifespan of tracking cookies. To ensure data privacy, maximize tracking accuracy, and keep user profiles unified over time, we strongly recommend routing your Data Activation tags through a Server GTM (ssGTM) container.

At the bottom of the tag configuration, expand the **Server GTM configuration** section:
1. Check **Send events to Server GTM**.
2. Enter your **Server container URL** (e.g., `https://metrics.yourdomain.com`). 

*Note: This requires the [Supermetrics Data Activation Client and Tag 🔗](https://github.com/orgs/Supermetrics-Professional-Services/repositories) to be installed and configured in your server container.*

***

### 💡 A Note for Reviewers & Developers
This template utilizes standard GTM Sandboxed JavaScript APIs. For security and compliance:
* It does **not** use wildcard script injections (`injectScript` is strictly limited to the trusted Audience Match and Orchestration Match endpoints).
* E-commerce and routing payloads are processed safely using `sendPixel` with robust callbacks.
* Local storage and cookies are managed exclusively for the `_sm_da_uuid` first-party identifier.