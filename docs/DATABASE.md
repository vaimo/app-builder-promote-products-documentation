# Database Schema

The application uses **App Builder DB** (`@adobe/aio-lib-db`), which provides a MongoDB-compatible document store provisioned per App Builder workspace. DB connectivity uses IMS access tokens rather than connection strings.

**Region:** `emea` (configured in `app.config.yaml` under `runtimeManifest.database.region`)

---

## Collections

### `users`

Populated automatically by the `customerSaveObserver` runtime action whenever a Magento customer record is saved. An upsert is performed on `magento_id`, so re-saves update the existing document.

| Field | Type | Description |
|---|---|---|
| `_id` | `string` | Internal App Builder DB document ID |
| `magento_id` | `number` | Magento `entity_id` |
| `email` | `string` | Customer email address |
| `first_name` | `string` | Customer first name |
| `last_name` | `string` | Customer last name |
| `subscribed` | `boolean` | Whether the customer is subscribed (`is_subscribed` from Magento) |
| `created_at` | `string` | ISO timestamp from Magento |

> Only customers with `is_subscribed: true` are meaningful targets for mailing, but all saved customers are synced regardless so subscription status changes are tracked.

---

### `lists`

Named groups for segmenting subscribers. Created and managed by the Magento administrator via the **User Lists** tab.

| Field | Type | Description |
|---|---|---|
| `_id` | `string` | Auto-generated UUID |
| `name` | `string` | Display name of the list |
| `is_default` | `boolean?` | Whether this is the default list |
| `created_at` | `string` | ISO timestamp |

---

### `user_list_membership`

Junction table (many-to-many) mapping users to lists.

| Field | Type | Description |
|---|---|---|
| `_id` | `string` | Auto-generated UUID |
| `user_id` | `string` | References `users._id` |
| `list_id` | `string` | References `lists._id` |
| `created_at` | `string` | ISO timestamp |

> When processing scheduled emails (`mail-process-scheduled`), the action fetches memberships for a given `list_id`, then resolves recipient emails from the `users` collection via `_id` matching.

---

### `newsletter_templates`

HTML email templates created manually or AI-generated via the **All Templates** or **Generate Template with AI** tabs.

| Field | Type | Description |
|---|---|---|
| `_id` | `string` | Auto-generated UUID |
| `title` | `string` | Display name / template label |
| `html_content` | `string` | Full HTML email body |
| `subject` | `string?` | Email subject line |
| `sender_name` | `string?` | Override sender name (falls back to SendGrid config) |
| `sender_mail` | `string?` | Override sender email (falls back to SendGrid config) |
| `created_at` | `string` | ISO timestamp |

> Templates are sorted by `created_at` descending when listed.

---

### `scheduled_emails`

Defines a pending (or completed) email delivery job: one template sent to one user list at a scheduled time.

| Field | Type | Description |
|---|---|---|
| `_id` | `string` | Auto-generated UUID |
| `template_id` | `string` | References `newsletter_templates._id` |
| `list_id` | `string` | References `lists._id` |
| `send_at` | `string` | ISO timestamp — target delivery time |
| `sent` | `boolean` | `false` until processed; `true` after delivery |
| `sent_at` | `string?` | ISO timestamp set when delivery completes |
| `created_at` | `string` | ISO timestamp |

> The processing action (`mail-process-scheduled`) filters for `sent=false` and `send_at <= now`. Once sent, `sent` is set to `true` and `sent_at` is recorded. There is currently no retry mechanism for partial failures.

---

## ID Handling

All `_id` values are generated using `crypto.randomUUID()` in the application code before insertion. The DB client normalises IDs to strings via `String(value)` for filter operations. This means `_id` is always a UUID v4 string, not a MongoDB ObjectId.

---

## Relationships Diagram

```
users ──────────────────────────── user_list_membership ─── lists
  │                                       │                    │
  │ _id                           user_id │ list_id            │ _id
  │◄──────────────────────────────────────┤                    │
  │                                       └───────────────────►│

newsletter_templates ─── scheduled_emails ──────────────────── lists
        │                       │                                │
        │ _id          template_id │ list_id                     │ _id
        │◄──────────────────────┤  └────────────────────────────►│
```

---

## Database Connection Pattern

```typescript
// Used in every action that touches the DB
async function getDbClient(params: ActionParams) {
    const token = await generateAccessToken(params);
    const db = await libDb.init({ token: token.access_token, region: params.DB_REGION });
    return db.connect();
}

// Always close the client in a finally block
let client;
try {
    client = await getDbClient(params);
    const collection = await client.collection('collection_name');
    // ... operations
} finally {
    if (client) await client.close();
}
```
