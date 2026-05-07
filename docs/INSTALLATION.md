# Installation & Configuration Guide

This guide covers everything you need to install and configure **Promote Products** from Adobe Exchange. No command-line tools or file editing are required — the entire setup happens through the Adobe Commerce Admin and Adobe Exchange.

---

## Prerequisites

### Adobe Commerce version

| Requirement | Minimum version |
|---|---|
| Adobe Commerce (PaaS or SaaS) | **2.4.5** |
| Adobe Commerce Admin UI SDK | **3.3.1** |
| Adobe Commerce Eventing module (`magento/commerce-eventing`) | **1.9.0** |

> **Admin UI SDK 3.x is required.** This app uses extension points that are not available in Admin UI SDK 1.x or 2.x. If you are on an older version, upgrade the module before installing.

### Third-party accounts

- An **OpenAI** account with an API key and sufficient credits
- A **SendGrid** account with:
  - An API key with at least **Mail Send** permissions
  - A verified sender identity (email address or domain)

### Adobe Commerce modules that must be enabled

Both of the following modules must be installed and enabled on your Commerce instance **before** installing this app:

1. **Admin UI SDK** (`magento/commerce-backend-sdk`) — version 3.3.1 or later
2. **Adobe Commerce Eventing** (`magento/commerce-eventing`) — version 1.9.0 or later

The sections below walk through enabling each one.

---

## Prepare your Commerce instance

### Step A: Install and configure Admin UI SDK

The Admin UI SDK allows this app to embed its management panel directly inside the Magento Admin.

Full reference: [Admin UI SDK documentation](https://developer.adobe.com/commerce/extensibility/admin-ui-sdk/)

1. **Install the module** via Composer on your Commerce instance:

   Open the [Magento Admin](https://experienceleague.adobe.com/en/docs/commerce-admin/start/admin/admin) and navigate to **System → Tools → Web Setup Wizard** or ask your Commerce administrator to run the module installation. The module is `magento/commerce-backend-sdk`.

2. **Enable the Admin UI SDK** in the Magento Admin:

   Navigate to **Stores → Configuration → Adobe Services → Admin UI SDK** and set **Enable Admin UI SDK** to **Yes**.

3. **Save the configuration** and clear the Magento cache:

   Navigate to **System → Cache Management** and click **Flush Magento Cache**.

4. **Verify** the module is active by navigating to **System → Extensions → Admin UI SDK**. The status should show as enabled.

> For detailed configuration options see the [Admin UI SDK configuration guide](https://developer.adobe.com/commerce/extensibility/admin-ui-sdk/configuration/).

---

### Step B: Install and configure Adobe Commerce Eventing

Promote Products listens for the `observer.customer_save_commit_after` Commerce event to automatically sync customer data. The Adobe Commerce Eventing module must be installed and configured to deliver events to Adobe I/O.

Full reference: [Adobe Commerce Eventing documentation](https://developer.adobe.com/commerce/extensibility/events/)

1. **Install the module.** The module is `magento/commerce-eventing`. Contact your Adobe Commerce administrator or Adobe account team if it is not already available in your instance.

2. **Configure the Adobe I/O connection** in Magento Admin:

   Navigate to **Stores → Configuration → Adobe Services → Adobe I/O Events** and provide:

   | Field | Value |
   |---|---|
   | **Adobe I/O Environment** | Select your environment (Production or Stage) |
   | **Adobe Organization ID** | Your Adobe IMS Org ID |
   | **Adobe I/O API Key (Client ID)** | From your Adobe Developer Console project |
   | **Adobe I/O Client Secret** | From your Adobe Developer Console project |
   | **Adobe Commerce Instance ID** | A unique label for this Commerce store |
   | **Adobe I/O Workspace** | Select the workspace matching your Exchange install |

3. **Create an event provider:**

   Navigate to **System → Events → Event Providers**. If no provider exists, click **Create Event Provider** and associate it with your Adobe I/O project.

4. **Subscribe to the required event:**

   Navigate to **System → Events → Subscribe to Events** and ensure the following event is subscribed:

   | Event name | Required |
   |---|---|
   | `observer.customer_save_commit_after` | ✓ |

   Set the **Fields** to `*` (all fields) to ensure the full customer payload is delivered.

5. **Verify event delivery:**

   Navigate to **System → Events → Event Logs**. After creating or updating a customer, you should see a successful delivery log entry for `observer.customer_save_commit_after`.

> For a complete setup walkthrough see the [Commerce Eventing getting started guide](https://developer.adobe.com/commerce/extensibility/events/).

---

## Step 1: Install from Adobe Exchange

1. Go to [Adobe Exchange](https://exchange.adobe.com/) and find **Promote Products**.
2. Click **Get** and follow the checkout flow.
3. In the **Adobe Developer Distribution** environment selector, choose the Adobe Commerce workspace to install into.
4. Click **Install**.

During installation the app will automatically:
- Provision the App Builder database
- Create all required collections (`users`, `lists`, `user_list_membership`, `newsletter_templates`, `scheduled_emails`)
- Register the `customer_save_commit_after` Commerce event subscription

You will see progress updates on screen. Installation typically completes within 2–3 minutes.

---

## Step 2: Configure the App

Once installed, open the **Magento Admin**, navigate to:

**System → App Management → Promote Products**

Enter the following values:

| Field | Description |
|---|---|
| **Commerce Base URL** | The base URL of your Adobe Commerce store (e.g. `https://your-store.com/`). Include the trailing slash. |
| **OpenAI API Key** | Your OpenAI secret API key (stored encrypted). |
| **OpenAI Model** | The OpenAI model to use for generation. Default: `gpt-4`. You can use any Chat Completions-compatible model (e.g. `gpt-4o`, `gpt-4-turbo`). |
| **SendGrid API Key** | Your SendGrid API key with Mail Send permissions (stored encrypted). |
| **SendGrid From Email** | The verified sender email address your emails will be sent from. |
| **SendGrid From Name** | The display name shown to email recipients. |

Click **Save** when done. All values take effect immediately — no redeployment needed.

> **Note:** API keys are stored encrypted by Adobe's configuration service. They are never stored in plain text.

---

## Step 3: Verify Customer Sync

Promote Products syncs customer data automatically whenever a customer record is saved in Commerce. To verify it's working:

1. Create or update a customer in **Customers → All Customers**.
2. Open the **Promote Products** panel in the Admin sidebar.
3. Navigate to the **User Lists** tab — you should see the customer appear in the **Users** view.

If the customer does not appear, check that Commerce Eventing is enabled and that the event provider is correctly configured. See the [Adobe Commerce Eventing documentation](https://developer.adobe.com/commerce/extensibility/events/) for setup guidance.

---

## Using the App

Access the app from the Magento Admin sidebar under **Promote Products → Promote Products**.

### User Lists

Organize your subscribers into named lists for targeted campaigns. You can:
- Create and delete lists
- Assign or remove individual users from lists
- View all synced subscribers

### Newsletter Templates

Manage reusable HTML email templates. Templates can be created manually using the built-in rich-text editor or generated automatically using AI (see below).

### Generate Template with AI

Select a product from your Commerce catalog, optionally choose a cart price rule, provide targeting context (event, audience, product link), and click **Generate**. The app will:

1. Generate structured promotional content via OpenAI (subject line, hero headline, body text, CTA, etc.)
2. Let you review and edit all fields
3. Optionally generate a product image via DALL-E
4. Let you preview the assembled email
5. Save it as a reusable template

### Scheduled Emails

Schedule a template to be sent to a specific list at a specific date and time. The app processes pending emails every minute and marks them as sent after delivery.

To create a scheduled email:
1. Go to the **Scheduled Emails** tab
2. Click **Add Scheduled Email**
3. Select a template, a list, and a send date/time
4. Click **Save**

### Generate Landing Page with AI

Select a product and generate a full promotional landing page. You can:
- Generate structured content first, then render to HTML
- Or generate HTML directly and extract the structured content later for editing
- Preview the result before publishing
- Publish directly to a **Magento CMS Page** via the Commerce REST API

---

## Troubleshooting

### Customers are not appearing in User Lists

- Confirm that **Adobe Commerce Eventing** is enabled and the `customer_save_commit_after` event is subscribed.
- Create or update a test customer to trigger a new event.

### Emails are not being sent

- Verify the **SendGrid API Key** has `Mail Send` permissions.
- Confirm the **SendGrid From Email** is a verified sender in your SendGrid account.
- Check that scheduled emails have a `send_at` time in the past (the processor only sends due emails).

### AI generation fails or returns errors

- Confirm the **OpenAI API Key** is valid and has available credits.
- If using a custom model name, verify it is supported by the Chat Completions API.

### Commerce data (products, price rules) not loading

- Confirm the **Commerce Base URL** is correct and includes the trailing slash.
- Verify that your Commerce instance is accessible from the internet (required for App Builder runtime actions to reach it).

---

## Required Adobe Commerce Permissions

The app accesses Adobe Commerce via IMS authentication. No manual integration credentials are required. The following Commerce APIs are used:

| API | Purpose |
|---|---|
| GraphQL `GetProducts` | Load product catalog in the app UI |
| REST `salesRules/search` | Load cart price rules for AI context |
| REST `cmsPage` (POST) | Publish AI-generated landing pages to CMS |
| Commerce Eventing | Receive `customer_save_commit_after` events |

---

## Data & Privacy

- Customer data (name, email, subscription status) is stored in the app's **App Builder database**, scoped to your Adobe organization and workspace.
- API keys entered in App Management are stored **encrypted** by Adobe's configuration service.
- No customer data is sent to OpenAI or SendGrid without explicit action by an administrator.
- Email delivery is handled by SendGrid using your account and credentials.
