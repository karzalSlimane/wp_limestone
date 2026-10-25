# Project Design Document — Moroccan Natural Stone B2B Website

> **Purpose of this file:** This is the single source of truth for the coding agent (Codex) and for the human architect. Every prompt, feature request, and code generation task must reference this document. If a requirement is not in this file, it is not part of the current scope.

---

## 1. Project Overview

**Business:** Marble and natural stone supply company specializing in Moroccan natural stone, with core products being **Taza Pierre (Taza Gray Limestone)** and **Taza Beige Limestone**.

**Website goal:** A B2B digital showroom and lead-generation engine that:
- Presents the full stone catalog with technical precision.
- Captures qualified inquiries (RFQs) from architects, contractors, importers, and distributors.
- Builds export credibility through trust signals (factory photos, packing, shipment history).
- Ranks for stone-specific searches (e.g., "Taza Gray limestone honed slabs").

**Primary audience:**
1. International importers and distributors of natural stone.
2. Architects and interior designers specifying materials.
3. Contractors and developers sourcing for projects.
4. Local (Moroccan) buyers and project owners.

**Primary conversion action:** Submit a product-specific B2B inquiry form (RFQ) or initiate a context-rich WhatsApp conversation.

**Secondary conversion action:** Download a technical spec sheet (PDF) for a specific stone.

**Languages (Phase 1):** English. **Phase 2:** French and Arabic.

---

## 2. Tech Stack

| Layer | Choice | Rationale |
|---|---|---|
| Framework | **Next.js (App Router)** | SEO metadata per stone page, image optimization, fast filtering, server components for catalog data. |
| Language | **TypeScript** | Type safety for the product catalog schema; reduces Codex errors. |
| Styling | **Tailwind CSS** | Rapid, consistent UI; easy for Codex to generate matching components. |
| UI Components | **shadcn/ui** | Accessible, unstyled primitives; customize to the stone brand. |
| Database | **PostgreSQL (Supabase)** | Managed Postgres; free tier sufficient for launch. |
| ORM | **Prisma** | Code-defined schema; Codex can read `schema.prisma` directly. |
| File/Image Storage | **Supabase Storage** or **Cloudinary** | Optimized image delivery for large stone photos (WebP, responsive sizes). |
| Forms | **React Hook Form + Zod** | Client validation; Zod schemas shared with server actions. |
| Email (inquiry notifications) | **Resend** | Simple transactional email for RFQ notifications to sales inbox. |
| Deployment | **Vercel** | Native Next.js support; preview deployments per branch. |
| Analytics | **Vercel Analytics + Google Search Console** | Traffic and SEO monitoring. |

**Do NOT introduce** additional frameworks, state managers, or UI libraries without updating this document first.

---

## 3. Database Schema (Prisma)

This is the authoritative schema. Codex must generate Prisma models exactly matching this structure.

```prisma
// schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

enum StoneType {
  LIMESTONE
  MARBLE
  TRAVERTINE
  GRANITE
  ONYX
}

enum ColorFamily {
  BEIGE
  GRAY
  WHITE
  BLACK
  BROWN
  MULTICOLOR
}

enum Finish {
  POLISHED
  HONED
  SANDBLASTED
  BUSH_HAMMERED
  AGED
  BRUSHED
  NATURAL_SPLIT
}

enum Format {
  BLOCK
  SLAB
  TILE
  CUT_TO_SIZE
  STAIR
  COUNTERTOP
  PAVER
  CLADDING
  MOSAIC
}

model Stone {
  id              String        @id @default(cuid())
  slug            String        @unique          // e.g. "taza-gray-limestone"
  name            String                         // e.g. "Taza Gray Limestone"
  shortDescription String                        // 1–2 sentence summary
  description     String        @db.Text         // full marketing + technical copy
  stoneType       StoneType
  colorFamily     ColorFamily
  originQuarry    String                         // e.g. "Taza region, Morocco"
  isFeatured      Boolean       @default(false)
  isActive        Boolean       @default(true)
  technicalSpecs  Json                           // { density, waterAbsorption, compressiveStrength, flexuralStrength, frostResistance }
  finishes        Finish[]
  formats         Format[]
  images          StoneImage[]
  inquiries       Inquiry[]
  createdAt       DateTime      @default(now())
  updatedAt       DateTime      @updatedAt

  @@index([stoneType, colorFamily])
}

model StoneImage {
  id         String  @id @default(cuid())
  stoneId    String
  stone      Stone   @relation(fields: [stoneId], references: [id], onDelete: Cascade)
  url        String
  altText    String
  isPrimary  Boolean @default(false)
  sortOrder  Int     @default(0)
}

model Inquiry {
  id             String   @id @default(cuid())
  stoneId        String?
  stone          Stone?   @relation(fields: [stoneId], references: [id])
  fullName       String
  company        String
  email          String
  phone          String?
  country        String
  projectType    String?              // e.g. "Hotel lobby flooring"
  requiredFormat Format?
  requiredFinish Finish?
  quantity       String?              // free text: "2000 sqm" or "1 container"
  message        String   @db.Text
  fileUrl        String?              // uploaded drawing / spec file
  status         String   @default("NEW") // NEW | CONTACTED | QUOTED | CLOSED
  createdAt      DateTime @default(now())
}

model Category {
  id          String   @id @default(cuid())
  slug        String   @unique
  name        String
  description String   @db.Text
  sortOrder   Int      @default(0)
}
```

---

## 4. Site Structure & Pages

### Public Routes

```
/                                 → Homepage
/stones                           → Full catalog with filters
/stones/[slug]                    → Individual stone detail page (e.g. /stones/taza-gray-limestone)
/categories/[slug]                → Curated category hub (e.g. /categories/limestone)
/materials-guide                  → Educational content (finishes, formats, applications)
/about                            → Company + quarry + factory story
/export                           → Shipping, packing, and export process
/contact                          → General contact + office info
/request-quote                    → Standalone RFQ form (for buyers who don't start from a product)
/privacy-policy
/terms
```

### Page Requirements

#### Homepage (`/`)
- **Hero:** High-quality full-width image of Taza stone in a real application; headline + subheadline; primary CTA "Explore Our Stones", secondary CTA "Request a Quote".
- **Featured Stones:** 4–6 stone cards pulled from `Stone` where `isFeatured = true`.
- **Why Choose Us:** 4 trust pillars (quarry-direct, consistent quality, export-ready packing, custom formats).
- **Applications section:** Flooring, cladding, stairs, countertops, paving — each with a representative image.
- **Export/Trust strip:** Factory photos, container loading photos, countries served.
- **CTA band:** "Request your project quote" with link to `/request-quote`.

#### Catalog (`/stones`)
- Server-rendered grid of all active stones.
- **Filters (URL-driven query params):** stone type, color family, finish, format.
- Each filter selection updates the URL (e.g. `/stones?color=GRAY&finish=HONED`) so results are shareable and SEO-indexable.
- Each card shows: primary image, name, color family, available finishes (as small tags), link to detail page.

#### Stone Detail (`/stones/[slug]`)
- Gallery (primary image large, thumbnails below).
- Name, color family, stone type, origin.
- Short description + full description.
- **Technical specs table** rendered from `technicalSpecs` JSON.
- Finish selector (visual chips) and format selector.
- **Product-specific inquiry form** — pre-fills `stoneId`, stone name, and shows a compact form (see Section 6).
- **WhatsApp button** — pre-filled message: `Hello, I'm interested in ${stone.name} (${canonicalUrl}). Could you share availability and pricing?`
- **"Request a sample"** secondary CTA (opens same form with `projectType = "Sample Request"`).
- Related stones (same color family or stone type).

#### Materials Guide (`/materials-guide`)
- Static educational content explaining: what is limestone vs. marble, what each finish means, what each format is used for, care and maintenance.
- Each finish and format linked from stone pages back to anchors here.

#### Export (`/export`)
- Packing process (photos of crates, containers).
- Shipping terms (FOB, CIF — define which you offer).
- FAQ: lead times, minimum order quantity, documentation.
- CTA to contact export team.

---

## 5. Component Architecture

Codex must generate components in `src/components/` organized by domain:

```
src/
  app/
    (public)/
      page.tsx                        → Homepage
      stones/
        page.tsx                      → Catalog
        [slug]/page.tsx               → Stone detail
      categories/
        [slug]/page.tsx
      materials-guide/page.tsx
      about/page.tsx
      export/page.tsx
      contact/page.tsx
      request-quote/page.tsx
      layout.tsx
    api/
      inquiries/route.ts              → POST handler: save inquiry + send email
      upload/route.ts                 → Optional file upload handler
  components/
    stones/
      StoneCard.tsx
      StoneGrid.tsx
      StoneGallery.tsx
      StoneSpecsTable.tsx
      StoneFilters.tsx
      FinishSelector.tsx
      FormatSelector.tsx
    forms/
      InquiryForm.tsx                 → Reusable, accepts optional stoneId
      RequestQuoteForm.tsx            → Standalone version
    layout/
      Header.tsx
      Footer.tsx
      WhatsAppButton.tsx
    ui/                               → shadcn/ui primitives
  lib/
    prisma.ts                         → Prisma client singleton
    validations/
      inquiry.ts                      → Zod schema for inquiry
    seo.ts                            → Metadata generators
    whatsapp.ts                       → WhatsApp link builder
  types/
    stone.ts                          → Shared TS types
```

**Component rules for Codex:**
- All components in TypeScript with explicit prop types.
- Server components by default; use `"use client"` only for interactive components (filters, forms, gallery lightbox).
- All images use `next/image` with explicit width/height or `fill` + `sizes`.
- No inline styles; Tailwind classes only.
- Forms use React Hook Form + Zod; server actions or API route for submission.

---

## 6. Inquiry Form Specification (Critical)

This is the core conversion element. Codex must implement it precisely.

### Fields

| Field | Type | Required | Notes |
|---|---|---|---|
| `stoneId` | hidden (from context) | No | Set when opened from a stone page. |
| `fullName` | text | Yes | |
| `company` | text | Yes | B2B qualification. |
| `email` | email | Yes | |
| `phone` | tel | No | |
| `country` | text or select | Yes | |
| `projectType` | select | No | Options: Residential, Commercial, Hospitality, Public, Sample Request, Other. |
| `requiredFormat` | select (Format enum) | No | |
| `requiredFinish` | select (Finish enum) | No | |
| `quantity` | text | No | Free text: "2000 sqm", "1 container", "50 slabs". |
| `message` | textarea | Yes | |
| `file` | file upload | No | Max 10 MB; accepted: PDF, DWG, JPG, PNG. |

### Behavior
- On stone page, the form shows a compact heading: "Request a quote for **[Stone Name]**".
- On `/request-quote`, the same component renders without a stone context.
- Successful submit → replace form with a success state: "Thank you. Our team will respond within 1 business day."
- On submit: create `Inquiry` record; send email via Resend to sales inbox; optionally send auto-reply to user.
- Server-side Zod validation mirrors client-side validation.

---

## 7. SEO Requirements

- **Metadata per page:** Use Next.js `generateMetadata` for stone pages — title: `"{Stone Name} — {Color} {StoneType} from Morocco | {CompanyName}"`; description from `shortDescription`.
- **Canonical URLs** on all pages.
- **Open Graph + Twitter cards** with stone primary image.
- **JSON-LD structured data:**
  - `Product` schema on each stone page (name, image, description, brand, offers optional).
  - `Organization` schema in root layout.
  - `BreadcrumbList` on catalog and stone pages.
- **Sitemap:** Generate `sitemap.xml` automatically from active stones + static pages.
- **`robots.txt`:** Allow all; disallow `/api/`.
- **Image alt text:** Every `StoneImage` must have `altText` set to `"{Stone Name} — {finish} finish, {format}"` at minimum.

---

## 8. Performance Requirements

- Lighthouse performance score **≥ 90** on mobile for homepage and stone pages.
- All images served as **WebP/AVIF** via `next/image` or Cloudinary.
- Lazy-load below-the-fold images and the gallery lightbox.
- LCP image on stone pages must be `priority` in `next/image`.
- No client-side JavaScript required to render the catalog or stone specs (server components).

---

## 9. Design & Brand Direction

- **Tone:** Professional, natural, material-focused, export-ready.
- **Color palette:** Neutral warm grays, off-whites, and a single accent (e.g., deep terracotta or forest green) for CTAs and links. Stone photography should dominate — do not compete with it.
- **Typography:** A clean sans-serif for UI (e.g., Inter) and a refined serif for headings (e.g., Cormorant Garamond or similar) to convey craft and quality.
- **Imagery:** Real photos only. No stock placeholder stone images in production. Use a named placeholder file during development.
- **Whitespace:** Generous. Stone needs breathing room.

---

## 10. Development Phases

### Phase 1 — Foundation (Weeks 1–4)
- Project scaffold (Next.js + Tailwind + shadcn/ui + Prisma + Supabase).
- Database schema migrated.
- Homepage, catalog, and stone detail pages with 2–4 real stones.
- Working `InquiryForm` with email delivery.
- WhatsApp button with pre-filled context.
- Deploy to Vercel on production domain.

### Phase 2 — Catalog Depth (Weeks 5–10)
- Full stone catalog populated.
- URL-driven filters working.
- Technical specs tables populated.
- Materials guide, about, export pages.
- SEO metadata, JSON-LD, sitemap complete.
- Google Search Console + Analytics connected.

### Phase 3 — Optimization & Expansion (Ongoing)
- French and Arabic translations.
- Technical spec PDF downloads.
- Blog for B2B trade content.
- Optional dealer/inventory portal.

---

## 11. Rules for the Coding Agent (Codex)

1. **Read this document before generating any code.** If a feature is not described here, ask before implementing.
2. **Follow the file structure in Section 5 exactly.** Do not introduce new top-level folders without updating this document.
3. **Every Prisma model change must be reflected in Section 3 of this document.** Update the doc in the same commit.
4. **No new dependencies** outside the tech stack in Section 2 without explicit approval.
5. **TypeScript strict mode is on.** No `any` unless unavoidable and commented.
6. **Every form field must have a Zod schema** and both client and server validation.
7. **Every page must export `metadata` or `generateMetadata`.**
8. **Generate one component or route at a time** when requested; do not scaffold the entire app in one response.
9. **Include a short comment block at the top of each generated file** stating its purpose and which section of this document it implements.
10. **When uncertain about stone-specific domain details** (finishes, formats, technical values), leave a clearly marked `// TODO: confirm with domain expert` rather than inventing values.

---

## 12. Open Questions (Resolve Before Phase 2)

- [ ] Which finishes are actually available for Taza Gray vs. Taza Beige? (Confirm with production team.)
- [ ] Which shipping terms do we offer internationally? (FOB / CIF / both?)
- [ ] What is the minimum order quantity (MOQ) per format?
- [ ] What are the standard container loading quantities for slabs and tiles?
- [ ] Do we have real factory, quarry, and container-loading photos for the Export page?
- [ ] What countries have we already shipped to? (Needed for trust section.)
- [ ] Primary sales email address and WhatsApp business number.

---

*Last updated: [DATE] — Owner: [YOUR NAME]*
