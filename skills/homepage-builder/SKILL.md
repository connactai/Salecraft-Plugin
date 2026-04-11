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

**Image Embed** (recommended):
```html
<div class="lp-embed-container">
  <div class="lp-embed" data-ratio="{aspect_ratio}">
    <img src="assets/stripe-0.webp" alt="{stripe_headline}" loading="lazy" width="1920" height="1080">
    <!-- Overlay CTA button -->
    <a href="{cta_link}" class="lp-cta-overlay">{cta_text}</a>
  </div>
</div>
```

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

Next: Open index.html in browser to preview, or proceed to /mx-publish
```

## Multiple LPs in One Homepage

For multi-product layouts:
1. Get data for each LP (repeat Phase 1 per campaign_id)
2. Create a grid/carousel of LP sections
3. Each LP card links to its full interactive version
