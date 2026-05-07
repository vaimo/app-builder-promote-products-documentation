# AI Generation

This document describes all AI-powered features in PromoteProducts — how they work, what they produce, and how the UI connects them.

---

## Configuration

All AI features use **OpenAI** (configured per store view via App Management):

| Config Key | Description |
|---|---|
| `openai-api-key` | OpenAI API key (stored encrypted) |
| `openai-model` | Model name; default `gpt-4` |

The configuration is read inside each AI runtime action via:
```typescript
const apiKeyConfig = await getConfigurationByKey("openai-api-key", byCodeAndLevel("default", "store_view"), { encryptionKey: params.AIO_COMMERCE_CONFIG_ENCRYPTION_KEY });
const modelConfig  = await getConfigurationByKey("openai-model",   byCodeAndLevel("default", "store_view"));
```

---

## Email Template Generation

### Flow

```
Admin UI (AiTemplates tab)
  │
  ├─ Select product (commerce-graphql)
  ├─ Select cart price rule (commerce-rest-get)
  ├─ Enter: event name, target audience, product link
  │
  ├─► [Generate] → ai-generate-promotion-text
  │         │
  │         └─► OpenAI Chat (GPT-4)
  │               System: "professional email marketing copywriter"
  │               User: dynamic prompt with product + context
  │               Response: JSON with 15 structured fields
  │
  ├─ Review / edit all fields in EmailContentEditor (Quill for rich text)
  ├─ Optionally: [Generate Image] → ai-generate-image → DALL-E
  ├─ [Preview] → assembled HTML email shown in PreviewDialog
  └─ [Save Template] → add-newsletter-template
```

### Prompt Construction

The prompt is built dynamically from user inputs:

```
"I want to promote our product {productName} [which has the following description: {productDescription}].
Can you please help me generate promotional email content...
[Mention this price rule as an advantage: {selectedRule}.]
[Time your advertising text to coincide with the {eventName}.]
[Target this email specifically for: {targetAudience}...]
[Include the product link {productLink} in the call-to-action.]
Please format your response as JSON with the following structure: {...}"
```

### Response Structure (`PromotionalEmailData`)

```ts
{
  subject_line:      string;   // Email subject
  preheader:         string;   // Preview text after subject
  hero_headline:     string;   // Main headline
  hero_subheadline:  string;   // Supporting headline
  body_intro:        string;   // Opening paragraph
  key_benefits:      string[]; // List of 3 benefits
  value_proposition: string;   // Main selling point
  call_to_action:    string;   // CTA button text
  urgency_text:      string;   // Scarcity/urgency copy
  closing_message:   string;   // Closing paragraph
  social_proof:      string;   // Testimonial/review snippet
  email_tone:        string;   // professional|friendly|urgent|casual
  target_audience:   string;   // Audience description
  product_link:      string;   // Echoes input
  product_name:      string;   // Echoes input
}
```

### Template Assembly

After generation (or editing), the UI assembles the final HTML using `assembleEmailTemplate()` in `web-src/src/components/tabs/AiTemplates/templateUtils.ts`. The assembled HTML is what gets saved to `newsletter_templates.html_content`.

---

## AI Image Generation

Accessible from the **Generate Template with AI** tab.

**Action:** `ai-generate-image`  
**Backend:** OpenAI image generation (DALL-E)

**Parameters:**
- `productName`, `productDescription` — describe what to generate
- `imageStyle` — style hint (e.g. "photorealistic", "minimalist")
- `referenceImageUrl` — optional reference for style/composition

**Response:** `{ imageUrl: string }` — URL of the generated image. The image URL can then be embedded into the email template HTML.

---

## Landing Page Generation

The **Generate Landing Page with AI** tab supports two generation paths:

### Path A: Structured Content First

```
[Generate Content] → ai-generate-landing-page-content
      │
      └─► OpenAI → LandingPageData JSON
            │
            ├─ Edit all sections in LandingPageContentEditor
            │     Sections: hero, value prop, problem/solution,
            │     key features, benefits, testimonials, pricing,
            │     urgency, guarantee, FAQ, final CTA
            │
            ├─ [Process / Regenerate HTML] → ai-regenerate-landing-page-html
            │       (sends edited JSON + existing HTML as context)
            │
            └─ [Preview] → PreviewDialog (rendered iFrame or innerHTML)
```

### Path B: HTML First

```
[Generate HTML] → ai-generate-landing-page-html
      │
      └─► OpenAI → Full HTML string
            │
            └─ [Extract Structured Data] → ai-extract-structured-data
                    │
                    └─► OpenAI parses HTML back to LandingPageData
                          └─ Then editable in LandingPageContentEditor
```

### Response Structure (`LandingPageData`)

```ts
{
  page_title, meta_description,
  hero_headline, hero_subheadline, hero_description, hero_cta,
  value_proposition, problem_statement, solution_description,
  key_features: string[],
  benefits_section: { title, description, benefits: string[] },
  testimonials: [{ name, role, text, rating }],
  social_proof_headline,
  pricing_section: { title, description, price_text },
  urgency_headline, urgency_text, guarantee_text,
  faq_section: { title, questions: [{ question, answer }] },
  final_cta_headline, final_cta_text, final_cta_button,
  product_link, product_name, product_image
}
```

### Publishing to Magento CMS Pages

From **CreatePageDialog**, the assembled landing page HTML is sent to Magento as a CMS page via:

```
commerce-rest-get  (operation: "cmsPage", body: { title, content, ... })
   │
   └─► POST {COMMERCE_BASE_URL}/rest/V1/cmsPage
```

This creates a new CMS page in Magento with the AI-generated HTML as the page content.

---

## Array Field Managers

Some sections (key features, testimonials, key benefits, product images) are arrays that need add/remove/update operations. These are managed by custom hooks:

| Hook | Location | Used in |
|---|---|---|
| `useKeyFeaturesManager` | `AiLandingPage/useArrayManagers.ts` | Landing page key features |
| `useTestimonialsManager` | `AiLandingPage/useArrayManagers.ts` | Landing page testimonials |
| `useKeyBenefitsManager` | `AiTemplates/useArrayManagers.ts` | Email key benefits |
| `useProductImagesManager` | `AiTemplates/useArrayManagers.ts` | Email product images |

---

## Rich Text Editing

User-editable rich text fields use **Quill** via `react-quill-new`. A known issue is that Quill emits console warnings in React 18 strict mode; these are suppressed via `suppressQuillWarnings()` called in `useEffect` at component mount.

---

## Error Handling in AI Actions

All AI action responses validate required fields before returning. For `ai-generate-promotion-text`, the required fields are: `subject_line`, `hero_headline`, `body_intro`, `call_to_action`. If any are missing from the OpenAI response, a 500 error is returned. JSON is extracted from the response using `indexOf('{')` / `lastIndexOf('}')` to handle cases where the model wraps JSON in markdown code blocks.
