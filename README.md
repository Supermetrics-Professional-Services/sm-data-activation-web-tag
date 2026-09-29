# Supermetrics Data Activation - Web Tag

This Google Tag Manager (GTM) template provides a seamless, script-free way to integrate your website with the Supermetrics Data Activation Customer Data Platform (CDP). 

Instead of loading a heavy custom JavaScript tracking snippet on your website, this template natively manages user identity, engagement tracking, e-commerce events, and real-time audience matching directly through GTM.

## 📋 Prerequisites
Before configuring your tags, you will need:
* **Your Site ID:** A unique number identifying your CDP instance (found in your Data Activation admin panel).
* **A Tracking Identifier (UUID):** A stable identifier for the user. We highly recommend using this template to generate a first-party `_svtri` cookie (see *Tag Types* below).

---

## 🏷️ Choosing Your Tag Type

This template is modular. You will create multiple tags in your GTM container using this single template by selecting different "Tag Types" from the dropdown.

### 1. Generate `_svtri` cookie (Identity Management)
**What it does:** This is the foundation of your tracking. It generates a unique, anonymous identifier (UUID) for new visitors and manages the browser cookie to ensure users are tracked consistently across sessions.
* **Best Practice:** Create one tag with this type and set it to fire on **All Pages** as early as possible (e.g., Initialization or Page View).

**Client-side cookie settings** — these two fields control the cookie written directly by the browser when no Server GTM URL is configured (or as a fallback when the server is unreachable):

* **Cookie Name *(optional)*:** The name of the first-party tracking cookie. Defaults to `_svtri`. You can override this if your organisation requires a different cookie name, for example to align with an existing naming convention.

* **Cookie Domain *(optional)*:** By default, browsers scope a cookie to the exact hostname where it was created — meaning a cookie set on `www.example.com` is invisible to `shop.example.com`. If your GTM container serves multiple subdomains on the same root domain, use this field to share a single tracking identity across all of them.

  Enter your **root domain with a leading dot** (e.g., `.example.com`). The tag will then set the cookie with this explicit `domain` attribute, making it readable on every subdomain — `www.example.com`, `shop.example.com`, `blog.example.com`, and so on — so that a visitor is recognised as the same person regardless of which subdomain they land on.

  > **When to use it:** Only needed when the same GTM container (or multiple containers sharing this tag) is deployed across two or more subdomains. If your tracking is confined to a single subdomain, leave this field blank and the browser's default scoping applies automatically.
  >
  > **Note:** This setting applies **only to client-side cookie writing**. When a Server GTM URL is configured and the server responds successfully, the cookie is written by the server via its own `Set-Cookie` header — this field has no effect on that server-set cookie. Cross-subdomain scope for server-set cookies is controlled by your Server GTM tag configuration.

* **Server GTM *(recommended)*:** To protect your tracking from Safari's Intelligent Tracking Prevention (ITP), which limits browser-written cookies to 7 days, expand the **Server GTM configuration** section and enter your **Server container URL**. This routes the cookie generation through your own Server GTM domain, granting it a stable, long-term lifespan. You will require to import our [`Supermetrics Data Activation - Server Client.tpl` 🔗](https://github.com/orgs/Supermetrics-Professional-Services/repositories) into your sGTM container to implement the server-set cookie creation.

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

### 6. Orchestration Match - Custom event
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
* Local storage and cookies are managed exclusively for the `_svtri` first-party identifier.