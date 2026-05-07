# Architecture

## System Architecture

PromoteProducts is an **Adobe App Builder** application that extends Adobe Commerce (Magento) using the platform's extensibility framework. It runs entirely on Adobe I/O Runtime (OpenWhisk) with no custom server infrastructure.

### High-Level Components

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Adobe Commerce (Magento)                      │
│                                                                      │
│  ┌──────────────────────┐      ┌──────────────────────────────────┐ │
│  │   Magento Admin UI   │      │       Commerce Events            │ │
│  │   (UIX extension)    │      │  observer.customer_save_         │ │
│  │                      │      │       commit_after               │ │
│  └──────────┬───────────┘      └─────────────┬────────────────────┘ │
└─────────────┼────────────────────────────────┼─────────────────────┘
              │                                │
              │ UIX API                        │ Adobe I/O Events
              ▼                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     Adobe I/O Runtime (OpenWhisk)                    │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                   PromoteProducts package                     │   │
│  │                                                              │   │
│  │  Web actions (Adobe Auth required):                          │   │
│  │   • commerce-graphql        • commerce-rest-get              │   │
│  │   • fetch/add/update/delete-newsletter-template              │   │
│  │   • fetch/add/delete-scheduled-email                         │   │
│  │   • fetch/add/remove-list   • assign/remove-user-*           │   │
│  │   • fetch-users             • fetch-user-list-membership     │   │
│  │   • ai-generate-promotion-text  • ai-generate-image          │   │
│  │   • ai-generate-landing-page-content                         │   │
│  │   • ai-generate-landing-page-html                            │   │
│  │   • ai-extract-structured-data                               │   │
│  │   • ai-regenerate-landing-page-html                          │   │
│  │   • mail-send                                                │   │
│  │                                                              │   │
│  │  Non-web actions (IMS credentials):                          │   │
│  │   • customerSaveObserver (triggered by Commerce event)       │   │
│  │   • mail-process-scheduled (cron / manual)                   │   │
│  │                                                              │   │
│  │  UIX registration action:                                    │   │
│  │   • admin-ui-sdk/registration                                │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌───────────────────────┐    ┌──────────────────────────────────┐  │
│  │  App Builder DB       │    │  Adobe Commerce Config Service   │  │
│  │  (MongoDB-compatible) │    │  (@adobe/aio-commerce-lib-config)│  │
│  │                       │    │                                  │  │
│  │  • users              │    │  Stores per store_view:          │  │
│  │  • lists              │    │  • openai-api-key (encrypted)    │  │
│  │  • user_list_         │    │  • commerce-base-url             │  │
│  │    membership         │    │  • openai-api-key (encrypted)    │  │
│  │  • newsletter_        │    │  • openai-model                  │  │
│  │    templates          │    │  • sendgrid-api-key (encrypted)  │  │
│  │  • scheduled_emails   │    │  • sendgrid-from-email           │  │
│  │                       │    │  • sendgrid-from-name            │  │
│  └───────────────────────┘    └──────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
              │                         │
              ▼                         ▼
    ┌──────────────────┐     ┌──────────────────────┐
    │    OpenAI API    │     │     SendGrid API      │
    │  (GPT-4 + DALL-E)│     │  (transactional mail) │
    └──────────────────┘     └──────────────────────┘
```

---

## Extension Points

The app registers three Adobe Commerce extension points in `app.config.yaml`:

### `commerce/backend-ui/1`
Adds two entry points in Magento Admin:
1. **Menu item**: _Promote Products > Promote Products_ — opens the main tabbed SPA
2. **Mass action**: _Promote Selected Products_ on the product grid (single product, opens a quick-generate popup)

### `commerce/configuration/1`
Provides the **App Management** configuration schema (`app.commerce.config.ts`) that allows Magento admins to store and update settings without editing environment variables.

| Field | Type | Description |
|---|---|---|
| `commerce-base-url` | `url` | Base URL of the Commerce instance (e.g. `https://your-store.com/`) |
| `openai-api-key` | `password` (encrypted) | OpenAI API key |
| `openai-model` | `text` | Model name, default `gpt-4` |
| `sendgrid-api-key` | `password` (encrypted) | SendGrid API key |
| `sendgrid-from-email` | `email` | Sender email address |
| `sendgrid-from-name` | `text` | Sender display name |

> **Commerce URL:** Adobe is planning to auto-provide the Commerce base URL in a future SDK release. Until then, it must be set manually in the App Management UI. The `commerce-rest-get` action reads it from `params['commerce-base-url']` with a fallback to `AIO_COMMERCE_API_BASE_URL` for forward compatibility.

### Authentication model

This app follows the Adobe App Management authentication model:

- **IMS (preferred):** On PaaS instances with the IMS module enabled, OAuth 1.0a Integration tokens are not required. The pre-app-build hook (`aio-commerce-lib-app hooks pre-app-build`) automatically injects IMS service account credentials. At runtime, `resolveCommerceHttpClientParams` from `@adobe/aio-commerce-lib-api` picks these up automatically.
- **Integration (fallback):** If IMS credentials are not present, `resolveCommerceHttpClientParams` falls back to OAuth 1.0a Integration credentials (`COMMERCE_CONSUMER_KEY`, etc.) for instances without the IMS module.
- **Forwarded IMS token:** For web actions, `tryForwardAuthProvider: true` allows forwarding the IMS token from the incoming `Authorization` header (proxy pattern).

### `commerce/extensibility/1`
Subscribes to the `observer.customer_save_commit_after` Commerce event, which fires when a Magento customer record is saved. The event payload is processed by `customerSaveObserver`.

---

## Frontend Architecture

The Admin UI SPA (`web-src/`) is built with:

- **React 18** + **React Router v6** (hash routing for UIX iframe compatibility)
- **Adobe React Spectrum** — design system components
- **Adobe UIX Guest SDK** (`@adobe/uix-guest`) — for the mass-action context (shared context: `selectedIds`)
- **React Error Boundary** — top-level error UI
- **Quill / react-quill-new** — rich text editor for email/landing page content

### Component tree

```
App
├── ExtensionRegistration    (UIX registration, loaded at root route)
└── PromoteProducts          (mass action, route: #/promote-products)
    └── NewsLetterManagement (main panel, route: # or sidebar link)
        ├── Tab 1: UserLists
        │     └── EditUsersModal (assign/remove users per list)
        ├── Tab 2: NewsletterTemplates
        │     └── EditTemplateModal
        ├── Tab 3: AiTemplates
        │     ├── EmailContentEditor
        │     ├── PreviewDialog
        │     └── CreateTemplateDialog
        ├── Tab 4: ScheduledEmails
        │     └── ScheduledEmailsModal
        └── Tab 5: AiLandingPage
              ├── LandingPageContentEditor
              │     ├── BenefitsSectionEditor
              │     ├── FAQSectionEditor
              │     ├── KeyFeaturesEditor
              │     ├── PricingSectionEditor
              │     ├── TestimonialsEditor
              │     └── RichTextEditor (Quill)
              ├── PreviewDialog
              └── CreatePageDialog (publish to Magento CMS)
```

### Data flow for UI actions

All frontend API calls go through `web-src/src/utils.ts → callAction()`, which:
1. Reads the action base URL from `config.json` (generated during build/deploy)
2. Attaches `x-gw-ims-org-id` and `Authorization: Bearer <ims_token>` headers
3. POSTs to the App Builder runtime action URL

---

## Action Architecture

All runtime actions are written in **TypeScript** in `actions-src/` and compiled to `dist/actions/` via `tsc`. The compiled output is what App Builder deploys.

### Core modules

| Module | Purpose |
|---|---|
| `aiActions.ts` | All OpenAI API calls — `generatePromotionText`, `generateLandingPageContent`, `generateLandingPageHTML`, `generateImageWithAI`, `extractStructuredDataFromHTML`, `regenerateLandingPageHTML` |
| `dbActions.ts` | All App Builder DB operations — template, scheduled email, list, user CRUD |
| `mailActions.ts` | SendGrid integration — `sendMail`, `processScheduledEmails` |
| `customerSaveObserver.ts` | Upserts Magento customer data to the `users` collection |

### Individual action entry points

Each file in `actions-src/ai/`, `actions-src/db/`, `actions-src/mail/`, etc. is a thin wrapper that:
1. Calls the corresponding function from the core module
2. Exports `main` in CommonJS format (required by OpenWhisk)

### Authentication patterns

| Pattern | Used by |
|---|---|
| `require-adobe-auth: true` | UI-facing actions — Adobe IMS token required in `Authorization` header |
| `include-ims-credentials: true` | Actions that need to generate an access token to call App Builder DB |
| `require-adobe-auth: false` | `commerce-rest-get` (uses OAuth 1.0a), `customerSaveObserver` (event-triggered) |

### Database access

The `@adobe/aio-lib-db` client connects via a generated IMS token (`generateAccessToken`). DB region is controlled by the `DB_REGION` environment variable (set to `emea` in `app.config.yaml`'s `database.region`).

---

## Scheduled Email Processing Flow

```
[OpenWhisk Alarm / manual invoke]
         │
         ▼
  mail-process-scheduled
         │
         ├─ fetch all scheduled_emails where sent=false and send_at <= now
         ├─ fetch all newsletter_templates
         ├─ fetch all users
         │
         └─ for each due email:
               ├─ resolve template by template_id
               ├─ resolve list members by list_id
               │     └─ fetch user_list_membership → users
               ├─ send HTML email via SendGrid to each subscriber email
               └─ mark scheduled_email as sent=true, sent_at=<timestamp>
```

---

## Configuration Management

App-level configuration (API keys) is managed via `@adobe/aio-commerce-lib-config`:

- **Write**: Magento Admin → App Management → PromoteProducts Extensibility
- **Read**: `getConfigurationByKey(key, byCodeAndLevel(storeCode, storeLevel), { encryptionKey })`
- Encrypted values (passwords) are decrypted at read time using `AIO_COMMERCE_CONFIG_ENCRYPTION_KEY`
