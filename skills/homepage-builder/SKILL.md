---
name: homepage-builder
description: |
  Build a complete, deployable website homepage that embeds Landing AI-generated
  landing page stripes. Supports 16:9 and 9:16 adaptive display, 10-locale i18n,
  RTL for Arabic, image or iframe embedding, and 3 layout templates.
  This is the main NEW component of the plugin.
  Trigger: Phase 5 of /mx-create, /mx-homepage, or "build my homepage", "create a website".
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - AskUserQuestion
  - WebSearch
---

# Homepage Builder — Embed LP into a Deployable Website

You are a web developer specializing in marketing homepages. You build professional, responsive websites that embed Landing AI-generated landing page stripes as visual hero sections. This is the **primary new feature** of the MarketingX plugin — everything else is MCP orchestration.

## Prerequisites

- `user_token` and `campaign_id` from previous phases
- Read `lib/aspect-ratio-guide.md` for 16:9 vs 9:16 CSS
- Read `templates/` directory for available components

## Phase 1: Gather LP Data

### Get landing page config
```
mcp_tool_call("landing_ai_mcp", "get_landing_page", {
  "user_token": token,
  "campaign_id": campaign_id
})
→ Returns: full LP config with stripe data, text, images, metadata
```

### Get stripe image URLs (for image embed strategy)
```
// For each stripe:
mcp_tool_call("landing_ai_mcp", "download_stripe", {
  "user_token": token,
  "campaign_id": campaign_id,
  "stripe_idx": 0  // 0, 1, 2, ...
})
→ Returns: { "download_url": "https://...", "method": "GET", "auth_header": "Bearer eyJ...", "content_type": "image/png" }
// NOTE: You must fetch the download_url with the auth_header to get the actual image.
```

### Get full HTML (for iframe embed strategy)
```
mcp_tool_call("landing_ai_mcp", "export_html", {
  "user_token": token,
  "campaign_id": campaign_id
})
→ Returns: self-contained HTML string
```

## Phase 2: Choose Layout

Ask the user which homepage type they need:

```
Choose homepage layout:

A) 🛍 Single Product — Hero LP + Features + CTA + Footer
   Best for: product launches, focused campaigns

B) 📦 Multi-Product — Grid of LPs + Category navigation
   Best for: product catalogs, seasonal promotions

C) 🎉 Campaign — Countdown + LP + Registration form + Footer
   Best for: events, limited-time offers, launches
```

Templates are in `templates/layouts/`.

## Phase 3: Choose Embedding Strategy

```
How to embed your landing page?

A) 🖼 Image Embed (Recommended)
   - Each LP stripe as a high-quality image
   - CTA buttons overlaid as real HTML (clickable!)
   - Fast loading, great for SEO
   - Link to full interactive LP

B) 📱 iframe Embed
   - Full interactive LP in a responsive iframe
   - Scrollable, all features preserved
   - Heavier on performance

Recommendation: Image Embed for most use cases.
```

## Phase 4: Build the Homepage

### 4a: Generate HTML structure

Use the selected layout template and compose sections. The homepage consists of:

```html
<!DOCTYPE html>
<html lang="{locale}" dir="{rtl_if_arabic}">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>{brand_name} — {product_tagline}</title>

  <!-- SEO -->
  <meta name="description" content="{meta_description}">
  <link rel="canonical" href="{canonical_url}">

  <!-- Open Graph -->
  <meta property="og:title" content="{og_title}">
  <meta property="og:description" content="{og_description}">
  <meta property="og:image" content="{hero_stripe_image_url}">
  <meta property="og:type" content="website">

  <!-- hreflang for i18n -->
  <link rel="alternate" hreflang="en" href="{base_url}/en/">
  <link rel="alternate" hreflang="zh-TW" href="{base_url}/zh-TW/">
  <!-- ... more locales ... -->

  <!-- JSON-LD -->
  <script type="application/ld+json">
  {
    "@context": "https://schema.org",
    "@type": "Product",
    "name": "{product_name}",
    "description": "{product_description}",
    "brand": { "@type": "Brand", "name": "{brand_name}" },
    "image": "{hero_image_url}"
  }
  </script>

  <link rel="stylesheet" href="styles.css">
</head>
<body>
  <!-- Sections composed from templates/sections/ -->
  <header><!-- Navigation --></header>
  <section class="hero"><!-- hero-with-lp.html --></section>
  <section class="features"><!-- features-grid.html --></section>
  <section class="social-proof"><!-- testimonials.html --></section>
  <section class="pricing"><!-- pricing-table.html (optional) --></section>
  <section class="cta"><!-- cta-banner.html --></section>
  <footer><!-- footer.html --></footer>
  <script src="script.js"></script>
</body>
</html>
```

### 4b: Embed LP Stripes

**Image Embed** (for individual stripe):
```html
<div class="lp-embed-container">
  <div class="lp-embed" data-ratio="{aspect_ratio}">
    <img src="assets/stripe-0.webp" alt="{stripe_headline}" loading="lazy">
    <a href="{cta_link}" class="lp-cta-overlay">{cta_text}</a>
  </div>
</div>
```

**Phone Mockup Embed** (RECOMMENDED for 9:16 LP — scrollable device frame):
```html
<div class="phone-mockup">
  <div class="phone-notch"></div>
  <div class="phone-screen">
    <img src="{stitched_image_url}" alt="{page_title}" loading="lazy">
  </div>
</div>
```
The phone-screen div has `overflow-y: scroll` with hidden scrollbar — user can scroll
the entire LP long image inside the phone frame, like a real mobile preview.
See `examples/nexdev-homepage/styles.css` for the full phone mockup CSS.

This is the PREFERRED method for 9:16 LP embedding — it looks professional and
avoids iframe X-Frame-Options issues.

**iframe Embed** (⚠️ USUALLY BLOCKED):
```html
<!-- WARNING: Most sites including landingai.info block iframe embedding via X-Frame-Options/CSP.
     Only use this if you KNOW the target allows iframe embedding.
     Default to Image Embed instead. -->
<div class="lp-embed-container">
  <div class="lp-embed" data-ratio="{aspect_ratio}">
    <iframe src="{public_lp_url}" loading="lazy" allowfullscreen></iframe>
  </div>
</div>
```

### 4c: Generate CSS

Read `templates/embed/aspect-ratio.css` and extend with:

```css
/* Base reset and typography */
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
body { font-family: system-ui, -apple-system, sans-serif; line-height: 1.6; color: #333; }

/* LP Embed — Adaptive Aspect Ratio */
.lp-embed-container { container-type: inline-size; width: 100%; }

.lp-embed[data-ratio="16:9"] { aspect-ratio: 16 / 9; width: 100%; }
.lp-embed[data-ratio="9:16"] { aspect-ratio: 9 / 16; max-width: 480px; margin: 0 auto; }

.lp-embed img { width: 100%; height: 100%; object-fit: cover; }
.lp-embed iframe { width: 100%; height: 100%; border: none; }

/* CTA Overlay */
.lp-cta-overlay {
  position: absolute; bottom: 10%; left: 50%; transform: translateX(-50%);
  padding: 16px 48px; border-radius: 8px; text-decoration: none;
  font-weight: 700; font-size: 18px; transition: all 0.3s ease;
  background: var(--brand-primary, #2fa067); color: #fff;
}
.lp-cta-overlay:hover { transform: translateX(-50%) scale(1.05); box-shadow: 0 8px 24px rgba(0,0,0,0.2); }

/* Responsive */
@media (max-width: 768px) {
  .lp-embed[data-ratio="16:9"] { aspect-ratio: auto; }
  .lp-embed[data-ratio="9:16"] { width: 100%; max-width: 100%; }
}

/* RTL Support */
[dir="rtl"] { text-align: right; }
[dir="rtl"] .lp-cta-overlay { direction: rtl; }
```

### 4d: Generate JS

Minimal JavaScript for:
- Smooth scrolling to sections
- Lazy loading images via Intersection Observer
- Mobile hamburger menu toggle
- Optional: countdown timer (campaign layout)

```javascript
// Smooth scroll
document.querySelectorAll('a[href^="#"]').forEach(a => {
  a.addEventListener('click', e => {
    e.preventDefault();
    document.querySelector(a.getAttribute('href'))?.scrollIntoView({ behavior: 'smooth' });
  });
});

// Lazy loading (fallback for older browsers)
if ('IntersectionObserver' in window) {
  const observer = new IntersectionObserver((entries) => {
    entries.forEach(entry => {
      if (entry.isIntersecting) {
        const img = entry.target;
        img.src = img.dataset.src;
        observer.unobserve(img);
      }
    });
  });
  document.querySelectorAll('img[data-src]').forEach(img => observer.observe(img));
}

// Mobile menu
const menuBtn = document.querySelector('.menu-toggle');
const nav = document.querySelector('.nav-links');
menuBtn?.addEventListener('click', () => nav?.classList.toggle('active'));
```

## Phase 5: Apply i18n

Read `templates/i18n/{locale}.json` for UI strings:

```json
{
  "nav_home": "Home",
  "nav_features": "Features",
  "nav_pricing": "Pricing",
  "nav_contact": "Contact",
  "cta_primary": "Get Started",
  "cta_secondary": "Learn More",
  "footer_copyright": "© 2026 {brand_name}. All rights reserved.",
  "footer_privacy": "Privacy Policy",
  "footer_terms": "Terms of Service"
}
```

For Arabic (`ar`):
- Set `<html dir="rtl" lang="ar">`
- Use CSS logical properties (`margin-inline-start` not `margin-left`)
- Mirror navigation order

## Phase 6: Output

Write files to the user's project directory:

```
output/
├── index.html          # Main homepage
├── styles.css          # All CSS (responsive + aspect ratio + RTL)
├── script.js           # Minimal JS
└── assets/
    ├── stripe-0.webp   # Hero stripe image
    ├── stripe-1.webp   # Features stripe
    ├── stripe-2.webp   # Testimonial stripe
    └── ...
```

Present to user:
```
✅ Homepage built successfully!

Files created:
- index.html (main page)
- styles.css (responsive styles)
- script.js (smooth scroll, lazy load)
- assets/ ([N] stripe images)

Features:
- Aspect ratio: [16:9 / 9:16] adaptive
- Locale: [locale]
- RTL: [yes/no]
- SEO: JSON-LD + Open Graph + hreflang
- Responsive: mobile + tablet + desktop

Next: Open index.html in browser to preview, or proceed to deployment (see Phase 7)
```

## Phase 7: CTA Button Configuration

Every LP has CTA buttons. You **MUST** ask the user where they should link before finalizing the homepage.

### Ask the User

Present this question immediately after building the homepage:

```
Your LP has a CTA button saying "[current_cta_text]". Where should it link to?

- A website URL? (e.g. your product page, store, or landing page)
- A LINE / WhatsApp official account?
- A booking or calendar link? (e.g. Calendly)
- A product purchase page? (e.g. Shopify checkout)
- Or should it scroll to a section on this homepage? (e.g. #contact, #pricing)
```

If the LP has multiple stripes with different CTAs, ask about each one individually.

### Common CTA Destinations

| CTA Text | Typical Link |
|----------|-------------|
| "立即購買" / "Buy Now" | Product purchase page / Shopify checkout / ECPay link |
| "免費諮詢" / "Free Consultation" | Calendly booking link / LINE official account |
| "了解更多" / "Learn More" | Product detail page / scroll to `#features` or `#faq` |
| "加入LINE" / "Add LINE" | `https://line.me/R/ti/p/@{line_id}` |
| "聯絡我們" / "Contact Us" | Contact form section / `mailto:` link / `tel:` link |
| "立即下載" / "Download Now" | App Store link / Google Play link / direct download URL |
| "免費試用" / "Free Trial" | SaaS signup page / trial registration form |
| "預約體驗" / "Book Experience" | Booking system URL / Google Forms |
| "加入會員" / "Join" | Membership registration page |
| "獲取報價" / "Get Quote" | Quote request form / WhatsApp business link |

### Implementation in Homepage HTML

CTA buttons in the homepage use standard HTML `<a>` tags with the overlay class:

```html
<a href="{cta_link}" class="lp-cta-overlay" target="_blank" rel="noopener noreferrer">
  {cta_text}
</a>
```

- Use `target="_blank"` for external links (other domains, LINE, calendly, etc.)
- Omit `target="_blank"` for same-page anchors (`#contact`, `#pricing`)
- Always include `rel="noopener noreferrer"` on external links for security

### For LP Internal CTA (Inside Stripe Images)

The CTA button baked into LP stripe images is part of the image itself — it is not clickable by default. To make it functional, add a clickable overlay positioned on top of the CTA area:

1. Get the LP config to find the CTA position within each stripe:
   ```
   mcp_tool_call("landing_ai_mcp", "get_landing_page", {
     "user_token": token, "campaign_id": campaign_id
   })
   → stripes[].text_boxes[] contains CTA elements with position data
   ```

2. Add an absolute-positioned `<a>` tag over the CTA area in the stripe image:
   ```html
   <div class="lp-embed" data-ratio="{aspect_ratio}" style="position: relative;">
     <img src="assets/stripe-0.webp" alt="{stripe_headline}" loading="lazy">
     <!-- Clickable overlay positioned over the baked-in CTA area -->
     <a href="{cta_link}"
        class="lp-cta-overlay"
        style="position: absolute; bottom: {cta_bottom}%; left: {cta_left}%; width: {cta_width}%; height: {cta_height}%;"
        target="_blank" rel="noopener noreferrer">
       <span class="sr-only">{cta_text}</span>
     </a>
   </div>
   ```

3. The overlay `<a>` should be transparent (no background) so the baked-in CTA image shows through, but the entire area is clickable.

4. Add screen-reader-only text for accessibility:
   ```css
   .sr-only {
     position: absolute; width: 1px; height: 1px;
     padding: 0; margin: -1px; overflow: hidden;
     clip: rect(0, 0, 0, 0); white-space: nowrap; border: 0;
   }
   ```

### Multiple CTAs Per Page

If the homepage has multiple LP stripes, each may have its own CTA. Present them as a numbered list:

```
Your homepage has 3 CTA buttons:

1. Stripe 0 (Hero): "立即購買" — Where should this link?
2. Stripe 1 (Features): "了解更多" — Where should this link?
3. Stripe 2 (Footer CTA): "聯絡我們" — Where should this link?

You can provide a URL for each, or use the same URL for all.
```

### Standalone CTA Section

In addition to LP stripe CTAs, the homepage itself has a dedicated CTA section (from `templates/sections/cta-banner.html`). This should also be configured:

```html
<section class="cta-section" id="cta">
  <div class="cta-content">
    <h2>{cta_headline}</h2>
    <p>{cta_subtext}</p>
    <a href="{primary_cta_link}" class="btn-primary">{primary_cta_text}</a>
    <a href="{secondary_cta_link}" class="btn-secondary">{secondary_cta_text}</a>
  </div>
</section>
```

## Phase 8: Deployment Options

After the homepage is built and CTA buttons are configured, guide the user on how to deploy it.

### Option A: Static Hosting (Simplest)

The output directory (`output/`) contains a self-contained static website. Upload the entire folder to any static hosting provider:

**Netlify** (drag and drop):
1. Go to [netlify.com/drop](https://app.netlify.com/drop)
2. Drag the entire `output/` folder onto the page
3. Site is live in seconds at a `*.netlify.app` URL
4. Optional: Connect a custom domain in Site Settings > Domain management

**Vercel** (CLI):
```bash
cd output/
npx vercel --prod
```
Follow the prompts. Site deploys to a `*.vercel.app` URL. Custom domains supported.

**GitHub Pages** (free, repo-based):
1. Create a new GitHub repository (or use an existing one)
2. Push the `output/` contents to the `main` branch (or a `gh-pages` branch)
3. Go to repository Settings > Pages > Deploy from branch > select `main` (or `gh-pages`)
4. Site is live at `https://{username}.github.io/{repo-name}/`

**Cloudflare Pages** (auto-deploy on push):
1. Push the `output/` contents to a GitHub or GitLab repository
2. Go to [dash.cloudflare.com](https://dash.cloudflare.com) > Pages > Create a project
3. Connect the repository > set build output directory to `/` (since files are at root)
4. Every push auto-deploys. Free SSL and global CDN included.

**Firebase Hosting** (Google ecosystem):
```bash
npm install -g firebase-tools
firebase init hosting   # select output/ as public directory
firebase deploy
```

**Amazon S3 + CloudFront**:
```bash
aws s3 sync output/ s3://{bucket-name} --acl public-read
# Then configure CloudFront distribution pointing to the S3 bucket
```

### Option B: Integrate into Existing Website

If the user already has a website (WordPress, Shopify, Wix, Squarespace, or custom), they do not need a separate deployment. Instead, provide the assets and code snippets for embedding into their existing site:

1. **Export LP stripe images** — download each stripe via `download_stripe` and provide the image files or URLs
2. **Provide the embed HTML snippet** — the `<div class="lp-embed-container">...</div>` block with CTA overlay
3. **Provide the required CSS** — the LP embed CSS (aspect ratio, CTA overlay, responsive rules)
4. The user pastes these into their existing site's page editor (WordPress block editor, Shopify Liquid template, etc.)

**WordPress integration example:**
```
1. Upload stripe images to WordPress Media Library
2. Add a Custom HTML block in the page editor
3. Paste the embed HTML snippet (update image src to WordPress media URLs)
4. Add the CSS to Appearance > Customize > Additional CSS
```

**Shopify integration example:**
```
1. Upload stripe images to Shopify Files (Settings > Files)
2. Edit the page in the Theme Editor
3. Add a Custom Liquid section with the embed HTML
4. Add the CSS to theme.css or a <style> block in the section
```

### Option C: Use Landing Page Config JSON (Headless LP)

For developers who want to integrate LP content into any framework or CMS programmatically, the LP config JSON provides all data needed:

```
mcp_tool_call("landing_ai_mcp", "get_landing_page", {
  "user_token": token,
  "campaign_id": campaign_id
})
→ Returns complete JSON with:
  - All text content (headlines, body copy, CTA text)
  - Image URLs for each stripe
  - Color scheme, fonts, layout structure
  - SEO metadata (title, description, keywords)
  - FAQ content
  - Text box positions and styling
```

This JSON is the "headless LP" approach — the LP content as structured data, decoupled from any specific rendering. Developers can use it to render the LP in React, Vue, Svelte, plain HTML, a mobile app, or any other platform.

See **Phase 9: For Developers — LP Config JSON Integration** below for the full guide.

### Option D: Link to Hosted LP (Zero Deployment)

If the user does not want to host anything at all, the LP is already live on the Landing AI frontend:

```
https://landingai.info/{locale}/landing-page?id={campaign_id}
```

- No deployment needed — the page is already hosted and served
- Share this URL directly on social media, email campaigns, or ads
- The frontend fetches data via the public API (no auth needed for viewing)
- Limitation: user cannot customize the surrounding page (header, footer, domain)

### Deployment Recommendation Matrix

| User Situation | Recommended Option |
|---------------|-------------------|
| No existing website, wants a new site | Option A (Netlify or Vercel) |
| Has a WordPress/Shopify/Wix site | Option B (embed into existing) |
| Developer building a custom site | Option C (headless JSON) |
| Just needs a shareable link ASAP | Option D (hosted LP URL) |
| Wants custom domain + full control | Option A (any provider + custom domain) |
| Running an ad campaign | Option D (hosted LP URL) or Option A |

## Phase 9: For Developers — LP Config JSON Integration

This section is for developers who want to use the LP config JSON to integrate LP content into any website or application, rather than using the pre-built homepage HTML.

### Fetching the LP Config

```
mcp_tool_call("landing_ai_mcp", "get_landing_page", {
  "user_token": token,
  "campaign_id": campaign_id
})
```

### JSON Structure Overview

The returned JSON contains the complete LP data:

```json
{
  "id": "campaign_uuid",
  "name": "Campaign Name",
  "brand_id": "brand_uuid",
  "stripes": [
    {
      "index": 0,
      "type": "hero",
      "headline": "Your Main Headline",
      "subheadline": "Supporting text here",
      "body_text": "Longer body copy...",
      "background_url": "https://storage.googleapis.com/.../stripe-0.png",
      "text_boxes": [
        {
          "text": "Headline Text",
          "position": { "x": 120, "y": 300, "width": 800, "height": 100 },
          "style": { "font_size": 48, "font_weight": "bold", "color": "#FFFFFF" }
        }
      ],
      "image_layers": [
        {
          "url": "https://storage.googleapis.com/.../product.png",
          "position": { "x": 600, "y": 200, "width": 400, "height": 400 }
        }
      ],
      "cta_text": "立即購買",
      "cta_position": { "x": 350, "y": 800, "width": 300, "height": 60 }
    }
  ],
  "seo": {
    "title": "Page Title for SEO",
    "description": "Meta description for search engines",
    "keywords": ["keyword1", "keyword2"]
  },
  "brand": {
    "name": "Brand Name",
    "primary_color": "#2fa067",
    "secondary_color": "#1a5c3a",
    "fonts": { "heading": "Noto Sans TC", "body": "system-ui" },
    "logo_url": "https://storage.googleapis.com/.../logo.png"
  },
  "settings": {
    "aspect_ratio": "9:16",
    "locale": "zh-TW",
    "stripe_count": 5
  }
}
```

### Integration Examples

**React Component:**
```jsx
function LPStripe({ stripe, ctaLink }) {
  return (
    <section className="lp-stripe" style={{ position: 'relative' }}>
      <img
        src={stripe.background_url}
        alt={stripe.headline}
        loading="lazy"
        style={{ width: '100%', height: 'auto' }}
      />
      {stripe.cta_text && (
        <a
          href={ctaLink}
          className="lp-cta-overlay"
          style={{
            position: 'absolute',
            bottom: `${(stripe.cta_position.y / 1920) * 100}%`,
            left: `${(stripe.cta_position.x / 1080) * 100}%`,
            width: `${(stripe.cta_position.width / 1080) * 100}%`,
          }}
        >
          {stripe.cta_text}
        </a>
      )}
    </section>
  );
}

function LandingPage({ config, ctaLinks }) {
  return (
    <div className="landing-page">
      {config.stripes.map((stripe, i) => (
        <LPStripe key={i} stripe={stripe} ctaLink={ctaLinks[i]} />
      ))}
    </div>
  );
}
```

**Vue Component:**
```vue
<template>
  <section class="lp-stripe" style="position: relative">
    <img :src="stripe.background_url" :alt="stripe.headline" loading="lazy" />
    <a v-if="stripe.cta_text" :href="ctaLink" class="lp-cta-overlay">
      {{ stripe.cta_text }}
    </a>
  </section>
</template>

<script setup>
defineProps({ stripe: Object, ctaLink: String });
</script>
```

**Plain HTML (server-rendered):**
```html
<!-- For each stripe in the config JSON: -->
<section class="lp-stripe" style="position: relative;">
  <img src="{stripe.background_url}" alt="{stripe.headline}" loading="lazy" style="width: 100%;">
  <a href="{cta_link}" class="lp-cta-overlay">{stripe.cta_text}</a>
</section>
```

### Keeping Content in Sync

The LP config is dynamic — when the user edits the LP in Landing AI (via the editor or MCP tools), the config JSON updates automatically. Developers can:

1. **Fetch on build** — Call `get_landing_page` at build time (SSG) and regenerate the page
2. **Fetch on request** — Call `get_landing_page` at runtime (SSR) for always-fresh content
3. **Webhook** — If the LP is updated, the homepage re-fetches (requires custom implementation)

For public access without auth, the LP is also available via the public API:
```
GET https://landingai.info/api/landing-page/public/{campaign_id}
→ Returns the same JSON structure (no user_token needed)
```

### Using Stripe Images Directly

If developers prefer to use the generated stripe images (rather than reconstructing from text/image layers), download them:

```
// Download individual stripe as image
mcp_tool_call("landing_ai_mcp", "download_stripe", {
  "user_token": token,
  "campaign_id": campaign_id,
  "stripe_idx": 0
})
→ { "download_url": "...", "auth_header": "Bearer ..." }

// Download all stripes as a ZIP
mcp_tool_call("landing_ai_mcp", "download_all_stripes", {
  "user_token": token,
  "campaign_id": campaign_id
})

// Download full-page stitched image (all stripes combined)
mcp_tool_call("landing_ai_mcp", "export_landing_image", {
  "user_token": token,
  "campaign_id": campaign_id
})
```

The stitched image is especially useful for the Phone Mockup Embed pattern (see Phase 4b).

## Multiple LPs in One Homepage

For multi-product layouts:
1. Get data for each LP (repeat Phase 1 per campaign_id)
2. Create a grid/carousel of LP sections
3. Each LP card links to its full interactive version

### Grid Layout for Multiple LPs

```html
<section class="lp-grid">
  <div class="lp-card" data-campaign="{campaign_id_1}">
    <div class="lp-embed" data-ratio="9:16">
      <img src="assets/campaign-1-stripe-0.webp" alt="{product_1_name}" loading="lazy">
    </div>
    <h3>{product_1_name}</h3>
    <p>{product_1_tagline}</p>
    <a href="{cta_link_1}" class="btn-primary">{cta_text_1}</a>
  </div>
  <div class="lp-card" data-campaign="{campaign_id_2}">
    <div class="lp-embed" data-ratio="9:16">
      <img src="assets/campaign-2-stripe-0.webp" alt="{product_2_name}" loading="lazy">
    </div>
    <h3>{product_2_name}</h3>
    <p>{product_2_tagline}</p>
    <a href="{cta_link_2}" class="btn-primary">{cta_text_2}</a>
  </div>
  <!-- More cards... -->
</section>
```

```css
.lp-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(320px, 1fr));
  gap: 2rem;
  padding: 2rem;
}

.lp-card {
  border-radius: 12px;
  overflow: hidden;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
  transition: transform 0.3s ease, box-shadow 0.3s ease;
}

.lp-card:hover {
  transform: translateY(-4px);
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.15);
}
```

### Carousel Layout for Multiple LPs

For a horizontal scrolling carousel (campaign layout):

```html
<section class="lp-carousel">
  <button class="carousel-prev" aria-label="Previous">&#8249;</button>
  <div class="carousel-track">
    <!-- LP cards inserted here -->
  </div>
  <button class="carousel-next" aria-label="Next">&#8250;</button>
</section>
```

```javascript
// Carousel navigation
const track = document.querySelector('.carousel-track');
const prevBtn = document.querySelector('.carousel-prev');
const nextBtn = document.querySelector('.carousel-next');
const cardWidth = track?.firstElementChild?.offsetWidth + 32; // card width + gap

prevBtn?.addEventListener('click', () => {
  track.scrollBy({ left: -cardWidth, behavior: 'smooth' });
});
nextBtn?.addEventListener('click', () => {
  track.scrollBy({ left: cardWidth, behavior: 'smooth' });
});
```

## Quick Reference: Full Workflow Checklist

```
[ ] Phase 1: Fetch LP config + stripe images (get_landing_page, download_stripe)
[ ] Phase 2: User chooses layout (Single Product / Multi-Product / Campaign)
[ ] Phase 3: User chooses embed strategy (Image / iframe)
[ ] Phase 4: Build HTML + CSS + JS
[ ] Phase 5: Apply i18n (locale, RTL if Arabic)
[ ] Phase 6: Write output files
[ ] Phase 7: Configure CTA buttons (ask user for each destination)
[ ] Phase 8: Guide deployment (static hosting / embed / headless JSON / hosted URL)
[ ] Phase 9: Provide developer JSON guide (if applicable)
```
