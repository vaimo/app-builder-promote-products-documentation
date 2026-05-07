# Runtime Actions Reference

All actions run on **Node.js 20** in the `PromoteProducts` OpenWhisk package unless noted. Source files are in `actions-src/` and compiled to `dist/actions/` by TypeScript.

---

## Commerce Integration Actions

### `commerce-rest-get`
**Source:** `actions-src/commerce/index.ts`  
**Auth:** `require-adobe-auth: false`, IMS credentials injected by pre-app-build

Proxy for Magento REST API calls. Uses `AdobeCommerceHttpClient` from `@adobe/aio-commerce-lib-api` with automatic auth resolution:
1. **IMS** — reads `AIO_COMMERCE_AUTH_IMS_*` credentials injected by the pre-app-build hook (PaaS with IMS module)
2. **Forwarded IMS token** — forwards the `Authorization` header from the incoming request
3. **OAuth 1.0a Integration** — fallback if IMS credentials are absent (non-IMS instances)

The Commerce base URL is read from the `commerce-base-url` business config field (set in App Management UI), with a forward-compatibility fallback to `AIO_COMMERCE_API_BASE_URL`.

**Parameters:**
| Param | Type | Required | Description |
|---|---|---|---|
| `operation` | `string` | ✓ | REST endpoint path, e.g. `salesRules/search?...` |
| `body` | `object` | — | POST body; only used when `operation` is `cmsPage` (POST) |

**Usage in UI:** Cart price rules retrieval; publishing CMS pages from the landing page generator.

---

### `commerce-graphql`
**Source:** `actions-src/graphql/index.ts`  
**Auth:** `require-adobe-auth: true`

Proxy for Adobe Commerce GraphQL queries. Executes the `GetProducts` query with pagination support. Returns an array of product objects including `uid`, `sku`, `name`, `url_key`, `description`, `image`, and `reviews`.

**Parameters (query string):**
| Param | Type | Default | Description |
|---|---|---|---|
| `pageSize` | `number` | — | Number of products per page |
| `currentPage` | `number` | — | Page number |

**Note:** The GraphQL endpoint is currently hardcoded to the playground Commerce instance. Update `COMMERCE_GRAPHQL_ENDPOINT` in `actions-src/graphql/index.ts` to point to your instance.

---

## Event Actions

### `customerSaveObserver`
**Source:** `actions-src/customerSaveObserver.ts`  
**Auth:** `require-adobe-auth: false`, `include-ims-credentials: true`  
**Trigger:** Commerce event `observer.customer_save_commit_after`

Upserts a customer record into the `users` collection when a Magento customer is saved. Uses `updateOne` with `upsert: true` matching on `magento_id`.

**Event payload expected at `params.data.value`:**
| Field | Mapped to |
|---|---|
| `entity_id` | `magento_id` |
| `email` | `email` |
| `firstname` | `first_name` |
| `lastname` | `last_name` |
| `is_subscribed` | `subscribed` |
| `created_at` | `created_at` |

---

## Mail Actions

### `mail-send`
**Source:** `actions-src/mail/send.ts`  
**Auth:** `require-adobe-auth: true`

Sends a single email via SendGrid. Reads `sendgrid-api-key`, `sendgrid-from-email`, and `sendgrid-from-name` from the Commerce Config service.

**Parameters:**
| Param | Type | Required | Description |
|---|---|---|---|
| `to` | `string \| string[]` | ✓ | Recipient email(s) |
| `subject` | `string` | ✓ | Email subject |
| `html` | `string` | ✓ | HTML email body |
| `text` | `string` | — | Plain-text fallback |

---

### `mail-process-scheduled`
**Source:** `actions-src/mail/processScheduled.ts`  
**Auth:** `require-adobe-auth: false`, `include-ims-credentials: true`  
**Trigger:** OpenWhisk alarm (cron) or manual invocation

Scans all `scheduled_emails` where `sent=false` and `send_at <= now`, resolves each to a template + list of recipient users, and sends via SendGrid. Marks each email as `sent=true` after successful delivery.

**No parameters required.** All configuration is read from Commerce Config and DB.

**Response:**
```json
{ "processed": 3, "results": [ { "_id": "...", "status": "sent", "recipientCount": 45 } ] }
```

> **Cron:** A trigger (`processScheduledEmailsTrigger`) and rule (`processScheduledEmailsRule`) are declared in `app.config.yaml` and deployed automatically with `aio app deploy`. The alarm fires every minute (`* * * * *`).

---

## Database Actions (CRUD)

All DB actions require `require-adobe-auth: true` and `include-ims-credentials: true`. They connect to App Builder DB using an IMS-generated access token.

### Newsletter Templates

| Action | Source | Operation |
|---|---|---|
| `fetch-newsletter-templates` | `db/fetchNewsletterTemplates.ts` | List all, sorted by `created_at` desc |
| `add-newsletter-template` | `db/addNewsletterTemplate.ts` | Insert new template |
| `update-newsletter-template` | `db/updateNewsletterTemplate.ts` | Update by `_id` |
| `delete-newsletter-template` | `db/deleteNewsletterTemplate.ts` | Delete by `_id` |

**Template document shape:**
```ts
{
  _id: string;          // auto-generated UUID
  title: string;
  html_content: string;
  subject?: string;
  sender_name?: string;
  sender_mail?: string;
  created_at: string;   // ISO timestamp
}
```

---

### Scheduled Emails

| Action | Source | Operation |
|---|---|---|
| `fetch-scheduled-emails` | `db/fetchScheduledEmails.ts` | List all |
| `add-scheduled-email` | `db/addScheduledEmail.ts` | Insert new scheduled job |
| `delete-scheduled-email` | `db/deleteScheduledEmail.ts` | Delete by `_id` |

**Scheduled email document shape:**
```ts
{
  _id: string;
  template_id: string;   // references newsletter_templates._id
  list_id: string;       // references lists._id
  send_at: string;       // ISO timestamp — when to send
  sent: boolean;         // default: false
  sent_at?: string;      // ISO timestamp — set after sending
  created_at: string;
}
```

---

### User Lists

| Action | Source | Operation |
|---|---|---|
| `fetch-user-lists` | `db/fetchUserLists.ts` | List all |
| `add-list` | `db/addList.ts` | Create new list |
| `remove-list` | `db/removeList.ts` | Delete by `_id` |

**List document shape:**
```ts
{
  _id: string;
  name: string;
  is_default?: boolean;
  created_at: string;
}
```

---

### Users & Memberships

| Action | Source | Operation |
|---|---|---|
| `fetch-users` | `db/fetchUsers.ts` | List all users |
| `assign-user-to-list` | `db/assignUserToList.ts` | Create membership record |
| `remove-user-from-list` | `db/removeUserFromList.ts` | Delete membership record |
| `fetch-user-list-membership` | `db/fetchUserListMembership.ts` | List all memberships |

---

## AI Generation Actions

All AI actions require `require-adobe-auth: true`. They read `openai-api-key` and `openai-model` from the Commerce Config service (encrypted).

### `ai-generate-promotion-text`
**Source:** `actions-src/ai/generatePromotionText.ts` → `aiActions.ts#generatePromotionText`

Generates structured promotional email content as JSON.

**Parameters:**
| Param | Type | Description |
|---|---|---|
| `productName` | `string` | Product name (required) |
| `productDescription` | `string` | Product description HTML |
| `selectedRule` | `string` | Cart price rule name |
| `eventName` | `string` | Seasonal event or occasion |
| `productLink` | `string` | Product URL for CTA |
| `targetAudience` | `string` | Audience description |

**Response `body.data`:**
```ts
{
  subject_line, preheader, hero_headline, hero_subheadline,
  body_intro, key_benefits: string[], value_proposition,
  call_to_action, urgency_text, closing_message,
  social_proof, email_tone, target_audience,
  product_link, product_name
}
```

---

### `ai-generate-image`
**Source:** `actions-src/ai/generateImageWithAI.ts` → `aiActions.ts#generateImageWithAI`

Generates a product promotional image via OpenAI image generation (DALL-E).

**Parameters:**
| Param | Type | Description |
|---|---|---|
| `productName` | `string` | Product name |
| `productDescription` | `string` | Product description |
| `imageStyle` | `string` | Style descriptor (e.g. "photorealistic") |
| `referenceImageUrl` | `string` | Optional reference image URL |

**Response `body.imageUrl`:** URL of the generated image.

---

### `ai-generate-landing-page-content`
**Source:** `actions-src/ai/generateLandingPageContent.ts` → `aiActions.ts#generateLandingPageContent`

Generates a structured `LandingPageData` object as JSON. Same parameters as `ai-generate-promotion-text`.

**Response `body.data`:** Full `LandingPageData` object (see [DATABASE.md](DATABASE.md) types or `web-src/src/components/tabs/AiLandingPage/types.ts`).

---

### `ai-generate-landing-page-html`
**Source:** `actions-src/ai/generateLandingPageHTML.ts` → `aiActions.ts#generateLandingPageHTML`

Generates a complete standalone HTML document for a landing page. Same parameters as above.

**Response `body.html`:** Full HTML string.

---

### `ai-extract-structured-data`
**Source:** `actions-src/ai/extractStructuredDataFromHTML.ts` → `aiActions.ts#extractStructuredDataFromHTML`

Parses an existing HTML landing page back into a `LandingPageData` structure. Useful for importing externally created pages into the editor.

**Parameters:**
| Param | Type | Description |
|---|---|---|
| `htmlContent` | `string` | HTML to parse |
| `productName` / `productDescription` / `productLink` | `string` | Context for the AI |

**Response `body.data`:** `LandingPageData` object.

---

### `ai-regenerate-landing-page-html`
**Source:** `actions-src/ai/regenerateLandingPageHTML.ts` → `aiActions.ts#regenerateLandingPageHTML`

Re-renders a full HTML landing page from a (user-edited) `LandingPageData` object and the previous HTML as context.

**Parameters:**
| Param | Type | Description |
|---|---|---|
| `editableLandingPageData` | `LandingPageData` | Edited structured data |
| `existingHtml` | `string` | Previous HTML (for style reference) |

**Response `body.html`:** Regenerated HTML string.

---

## UIX Registration Action

### `admin-ui-sdk/registration`
**Source:** `actions-src/registration/index.ts`  
**Auth:** `require-adobe-auth: true`

Returns the UIX registration payload that tells Magento Admin:
- **Mass action**: "Promote Selected Products" on the product grid (single product selection; opens `#/promote-products`)
- **Menu section**: "Promote Products" in the Admin sidebar
- **Menu item**: "Promote Products" under the section (opens the main SPA)
- **Page title**: "Promote Products"
