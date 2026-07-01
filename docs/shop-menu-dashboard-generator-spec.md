# Shop Menu Generator — Full Specification & Context

> **Read this first.** You (Claude) have **no prior knowledge** of Venn Apps, its dashboard, or its data model. This document contains **all** the context you need. Everything below — the schema, the rules, the real code, the real examples — is embedded here. There is no other repo or doc to look at.
>
> **Your job:** Build a tool that takes a menu *design* (a Figma export, screenshot(s) of a shop's menu, or a link to the customer's website) plus a target `storekey`, and produces **paste-ready JSON** that a human copies into the Venn Store Editor file called **"Shop Menus - Dashboard only"**.
>
> **You do the thinking.** This spec defines the *output format*, *rules*, and *data model*. It does not prescribe the extraction/OCR/scraping logic or the prompt design — that's yours to build.

---

## 0. Background context (what all this means)

**Venn Apps** builds native iOS/Android shopping apps for Shopify merchants ("clients" / "stores"). Each store's configuration lives in a database (Firestore). One of those config documents is the **shop menu** — the navigable category menu users tap through in the app (e.g. `MEN → HOODIES`, `WOMEN → DRESSES`).

Internally the team calls this document **"Shop Menus - Dashboard only"**. It is a JSON document. Implementation staff edit it and paste JSON into it via an internal tool called the **Store Editor**.

Today, building this menu by hand is slow: staff manually create each menu row, type a title, and link it to a Shopify **collection**. Duplicating a "MEN" tab into a "WOMEN" tab means re-doing everything because the collection IDs differ. **This tool automates that**: read the menu structure from a design/site, resolve each item to a collection, and emit the JSON.

Key vocabulary:

| Term | Meaning |
|------|---------|
| **Store / client** | A merchant's app configuration |
| **storekey** | The store's unique ID (a UUID), e.g. `5c116374-762f-4b37-bd63-c4749654c33a` |
| **Collection** | A Shopify product grouping (like "Men's Hoodies"). Has a **numeric ID** (e.g. `159983206452`) and a **handle** (e.g. `mens-hoodies`). Menus link to collections by **numeric ID**. |
| **Category** | Venn's synced copy of a Shopify collection. Same numeric ID. This is what you resolve against. |
| **Shop menu** | The nested tree of tappable menu items. This is what you generate. |
| **Store Editor** | Internal tool where staff paste the JSON |
| **Dashboard / CMS** | The merchant-facing UI (drag-and-drop). Same underlying data, different editor. Not your concern. |

---

## 1. Where the document lives & what it is

| Property | Value |
|----------|-------|
| Storage | Firestore collection `menus`, document ID = `{storekey}` |
| Published (live) path | `menus/{storekey}` |
| Draft (unpublished) path | `menus/{storekey}/drafts/CURRENT` |
| Regional/scheduled | `menus/{storekey}/versions/{versionId}` |

**"Shop Menus - Dashboard only" IS this `menus/{storekey}` document.**

**It is NOT:**
- The Shopify Admin navigation menu (different system, different shape with `handle`/`parentId`)
- The Dashboard CMS drag-and-drop editor (that's a UI over the same data)
- `infoConfig.shopMenu` — that is the *processed output* the mobile app receives after transformation (see §7). **Do not generate that shape.** You generate the raw dashboard `menu` array.

---

## 2. The exact output you must produce

The whole document looks like this:

```json
{
  "menu": [ /* array of MenuItem — THIS is what you build */ ],
  "theme": { /* optional global styling — omit unless asked */ },
  "conditionalMenus": { /* optional regional/tag variants — omit unless asked */ },
  "storekey": "5c116374-762f-4b37-bd63-c4749654c33a",
  "timestamp": 1735689600000
}
```

**Your primary deliverable is the `menu` array** — a `MenuItem[]`.

### Default output = the `menu` array only

Most of the time staff paste just the array into the existing document's menu field:

```json
[
  {
    "image": "",
    "title": "MEN",
    "link": { "type": "category", "payload": "" },
    "children": [ /* ... */ ]
  }
]
```

Also offer a **full-document** variant (the object in §2 top) for when the whole doc is replaced. Note: `storekey` inside the JSON is **optional** — many legacy docs omit it and the Store Editor knows the store from context. If you include it, use the real target storekey.

Format: pretty-printed JSON, 2-space indent.

---

## 3. The data model (embedded — this is the real source code)

### 3.1 `MenuItem` and `MenuDocument`

Verbatim from Venn's `MenuItem.ts`:

```typescript
export interface MenuItem {
  image: string;                          // REQUIRED. Use "" when no image (runtime backfills from collection image).
  title: string;                          // REQUIRED. The label shown in the app. May be "".
  link: VennLink;                         // REQUIRED. See 3.2.
  children: MenuItem[];                   // REQUIRED. Use [] for a leaf. Max depth 3 (see §5).
  translations?: Record<string, string>; // OPTIONAL. e.g. { "es": "Hombre" }. Dashboard-only; stripped before app. Omit unless doing multi-language.
  theme?: {                               // OPTIONAL per-item styling. Omit unless explicitly needed.
    isEnabled?: boolean;
    heightMultiplier?: number;
    selectedTitleTheme?: Partial<LabelStyle>;
    unselectedTitleTheme?: Partial<LabelStyle>;
    selectedIcon?: MenuGlobalIconTheme;
    unselectedIcon?: MenuGlobalIconTheme;
  };
}

export interface MenuDocument {
  menu: MenuItem[];
  theme?: MenuGlobalTheme;              // optional global menu theme
  conditionalMenus?: ConditionalMenus; // optional regional / customer-tag menus
  storekey?: string;                   // "Should not be optional, but many legacy docs don't have it."
  timestamp: number;
  storeEditorLastUpdated?: number;
  dashboardLastUpdated?: number;
  dashboardLastUpdatedSessionId?: string;
}

// Optional — only relevant if generating regional/tag-conditional menus (advanced, usually skip):
export type ConditionalMenus = {
  customerTags?: ConditionalMenuObject;
  countryCodes?: ConditionalMenuObject;
};
export type ConditionalMenuObject = Record<
  string,
  { metadata: { active: boolean }; menu: MenuItem[]; theme?: MenuGlobalTheme }
>;
```

### 3.2 `VennLink` (the `link` field)

Verbatim from Venn's `Homepages.ts`. **This same link type is used for both menu items and shop links.**

```typescript
export type PayloadType =
  | "empty"
  | "product"
  | "category"
  | "fallbackWebUrl"
  | "payloadAndroid"
  | "deeplink"
  | "tabBarId";

export interface VennLink {
  payload: string;             // REQUIRED. For a collection: the numeric collection ID as a string.
  type: PayloadType;           // REQUIRED. See §4 for which to use.
  readableTitle?: string;      // Human-readable collection title. STRONGLY recommended on category links.
  image?: ImageStyle;          // optional
  fallbackWebUrl?: string;     // optional
  payloadAndroid?: string;     // optional
  title?: string;              // optional
  requiredTag?: string;        // optional
  iconUrl?: string;            // optional
  blogTitle?: string;          // optional
  tabIndex?: number;           // optional
  scrollIndex?: number;        // optional
  showDetails?: boolean;       // optional
  routingMethod?: "push" | "modal" | "contextual" | "replace"; // optional
}
```

> **Note:** The *save-time validation* (below) allows a slightly different set of `type` values than the TypeScript `PayloadType`. Follow the **validation schema in §4/§5** as the authoritative allow-list, because that's what the Store Editor enforces on save. In practice you will almost always use only `"category"`, `"page"`, and `"empty"`.

---

## 4. Link types — which to use

The Store Editor validation (`menuSchema.ts`) allows exactly these `link.type` values:

```
"product" | "category" | "empty" | "" | "nativeWebPage" | "brands" | "tabBarId" | "page"
```

| `link.type` | Use for | `payload` |
|-------------|---------|-----------|
| `"category"` | **A collection link** (the vast majority of items). | **Numeric Shopify collection ID as a string**, e.g. `"159983206452"`. Empty `""` allowed on a container tab that has no direct link. |
| `"page"` | A **section header** row that just expands to show its children (no navigation itself). | `""` |
| `"empty"` | A placeholder with no link. | `""` |
| `"tabBarId"` | Jump to another tab bar item. Occasionally used on grouping rows. | tab identifier or `""` |
| `"product"` | Link straight to one product's PDP. | product ID string |
| `"nativeWebPage"` | Open an in-app webview at a URL. | URL string |
| `"brands"` | Open the brands screen. | per store |
| `""` | Legacy empty. Prefer `"empty"`. | `""` |

**Rules that matter most:**
1. **Collection links use the numeric collection ID in `payload`, never the handle.** `"159983206452"` ✅, `"mens-hoodies"` ❌.
2. **Always set `readableTitle`** on `"category"` links to the exact collection title. It's used for search, clone-to-new-store migration, and human debugging. It is **not** sent to the app at runtime, so it's safe and helpful to include.
3. A **container tab** (level 1 grouping like MEN) commonly uses `"type": "category"` with `"payload": ""` and just holds children.
4. A **section header** (a grouping row *inside* a tab, like "CATEGORY") uses `"type": "page"` with `"payload": ""`.

---

## 5. Nesting rules (hard limit: 3 levels)

```
Level 1  — Top tab / root          e.g. MEN, WOMEN, KIDS, SALE
  Level 2 — Section OR direct link  e.g. NEW IN (link) or CATEGORY (section header)
    Level 3 — Leaf collection link  e.g. HOODIES, JOGGERS   ← children MUST be []
```

The save validation (`menuSchema.ts`) enforces this. Verbatim:

```typescript
const menuLink = Joi.object({
  type: Joi.string()
    .valid("product","category","empty","","nativeWebPage","brands","tabBarId","page")
    .required(),
  payload: Joi.string().allow("").required(),
  readableTitle: Joi.string().allow(""),
});

const menuItemBase = {
  image: Joi.string().allow("").required(),
  title: Joi.string().allow("").required(),
  link: menuLink.required(),
  translations: multiLanguageMenuSchema,   // optional
  theme: /* optional per-item theme */,
};

// Explicitly written out to 3 levels — deeper nesting is rejected.
export const menuItemSchema = Joi.object({
  ...menuItemBase,
  children: Joi.array().items(
    Joi.object({
      ...menuItemBase,
      children: Joi.array().items(
        Joi.object({
          ...menuItemBase,
          children: Joi.array().length(0).required(),  // level 3 MUST have []
        })
      ).required(),
    })
  ).required(),
});
```

**Absolute rules:**
1. Max depth = **3**.
2. **Level-3 items must have `"children": []`.**
3. Every node must include all four keys: `image`, `title`, `link`, `children`.
4. `link.type` and `link.payload` are required on every node (payload may be `""`).

---

## 6. Real, complete example (Couture Club — a live store)

This is a **real** shop menu document (UAT). It shows the three structural patterns you'll produce: a **container tab** (MEN), **direct-link children**, **section headers** (`type: "page"`), a `tabBarId` grouping row, and an **empty tab** (WOMEN, before it was populated). Truncated for length but structurally complete.

```json
{
  "menu": [
    {
      "image": "",
      "title": "MEN",
      "link": { "type": "category", "payload": "" },
      "children": [
        {
          "image": "",
          "title": "NEW IN",
          "link": {
            "type": "category",
            "payload": "646118244728",
            "readableTitle": "MENSWEAR NEW IN PRIVATE"
          },
          "children": []
        },
        {
          "image": "",
          "title": "BESTSELLERS",
          "link": {
            "type": "category",
            "payload": "160526729268",
            "readableTitle": "Men's Clothing Bestsellers"
          },
          "children": []
        },
        {
          "image": "",
          "title": "CATEGORY",
          "link": { "type": "page", "payload": "", "readableTitle": "" },
          "children": [
            {
              "image": "",
              "title": "ALL MENSWEAR",
              "link": {
                "type": "category",
                "payload": "668001108344",
                "readableTitle": "ALL PRODUCTS"
              },
              "children": []
            },
            {
              "image": "",
              "title": "HOODIES",
              "link": {
                "type": "category",
                "payload": "159983206452",
                "readableTitle": "Men's Hoodies"
              },
              "children": []
            },
            {
              "image": "",
              "title": "JOGGERS",
              "link": {
                "type": "category",
                "payload": "669300818296",
                "readableTitle": "SALE MEN'S JOGGERS"
              },
              "children": []
            }
          ]
        },
        {
          "image": "",
          "title": "FOOTWEAR",
          "link": {
            "type": "category",
            "payload": "668654731640",
            "readableTitle": "Men's Luxury Trainers"
          },
          "children": []
        },
        {
          "image": "",
          "title": "ACCESSORIES",
          "link": { "type": "tabBarId", "payload": "", "readableTitle": "" },
          "children": [
            {
              "image": "",
              "title": "CAPS & BEANIES",
              "link": {
                "type": "category",
                "payload": "667793260920",
                "readableTitle": "Men's Caps and Beanies"
              },
              "children": []
            },
            {
              "image": "",
              "title": "VIEW ALL",
              "link": {
                "type": "category",
                "payload": "666742391160",
                "readableTitle": "MEN'S ACCESSORIES VIEW ALL"
              },
              "children": []
            }
          ]
        },
        {
          "image": "",
          "title": "SALE",
          "link": { "type": "category", "payload": "" },
          "children": [
            {
              "image": "",
              "title": "SHOP ALL",
              "link": {
                "type": "category",
                "payload": "262448676916",
                "readableTitle": "Men's Sale"
              },
              "children": []
            }
          ]
        }
      ]
    },
    {
      "image": "",
      "title": "WOMEN",
      "link": { "type": "category", "payload": "" },
      "children": []
    }
  ]
}
```

### Default template for a brand-new store (verbatim from `defaultMenus.ts`)

```json
{
  "menu": [
    {
      "image": "",
      "title": "Sample Menu Item",
      "link": { "type": "empty", "payload": "" },
      "children": []
    }
  ],
  "storekey": "<storekey>",
  "timestamp": 1
}
```

---

## 7. What happens to this JSON later (so you understand the format, don't replicate it)

Your JSON is stored raw. When the mobile app requests config, Venn's backend transforms it into `infoConfig.shopMenu`. **You do not produce the transformed shape** — but knowing the transform explains some conventions.

Verbatim behaviour from `ShopMenuTransformer.ts`:

1. **Empty/invalid links become `category`.** `if (link.type === "empty" || (!payload && !type)) link.type = "category"`. This is why container tabs can safely use `category` + empty payload.
2. **Missing images are backfilled** from the collection's image at runtime, keyed by the collection ID. This is why `"image": ""` is fine — you don't need to find images.
3. **`translations` are applied then stripped** before the app sees them.
4. **Only 3 levels are walked** (`transformShopMenuRecursively` is hand-written to exactly 3 depths). Reinforces the hard depth limit.

Implication for you: leave `image` as `""` (unless you have a real URL), always populate `readableTitle` on category links, and never exceed 3 levels.

---

## 8. Resolving menu labels → collection IDs (needs the target store)

Each leaf item must end up with a **numeric collection ID** in `payload`. You get those from the target store's category list.

### If running inside Cursor with Venn MCP available

Venn provides an MCP server ("VennMCP") with these relevant tools (real tool signatures):

```
list_venn_categories({ storekey, limit: 500, environment: "uat" | "production" })
  → { categories: [ { id, name, handle, description, isAvailable } ] }
  // id   = numeric collection ID string (goes in link.payload)
  // name = collection title (goes in link.readableTitle)
  // handle = collection handle (useful if you scraped handles/URLs)

get_venn_category({ storekey, categoryId, environment })
  → one category's detail

get_document({ storekey, collection: "menus", variant: "published" | "draft", environment })
  → the current menu document (use to merge or reference existing structure)
```

There is **no dedicated category text-search MCP tool** — pull the full list once with `list_venn_categories` (limit up to 500) and match locally.

### Resolution algorithm (recommended)

1. Fetch all categories for the target store once; cache them.
2. For each **leaf** label extracted from the design:
   - Normalize both sides: trim, lowercase, collapse whitespace.
   - Match in priority order:
     1. Exact match on `category.name`
     2. Fuzzy/contains match on `category.name` (or small edit distance)
     3. Match `category.handle` if you extracted a handle/URL slug from the website
   - On success: `payload = category.id`, `readableTitle = category.name`.
   - On failure: `payload = ""`, `readableTitle = "<the label you searched>"`, and **record a warning** so the human can fix it.
3. **Collections are store-specific.** When duplicating MEN → WOMEN, re-resolve each women's label against the store's women's collections — do **not** reuse the men's IDs.

How Venn's own clone tool does this (for reference): it reads each item's `readableTitle`, searches the new store's categories by that title, and swaps in the new ID; unmatched titles are blanked and flagged. Mirror that behaviour.

---

## 9. Turning a design/site into a structure (you design this part)

Produce an intermediate tree first, then resolve collections, then emit JSON.

Suggested intermediate type:

```typescript
interface ExtractedMenuNode {
  title: string;                 // label exactly as shown in design/site
  kind: "tab" | "section" | "leaf";
  children: ExtractedMenuNode[];
  collectionHandle?: string;     // optional hint if scraped from a website URL/slug
  collectionUrl?: string;        // optional hint
}
```

Heuristics for `kind`:

| Signal in design/site | kind |
|---|---|
| Top-level horizontal tab (MEN / WOMEN / KIDS) | `tab` (level 1) |
| Grouping label with indented items beneath, no own destination | `section` (`link.type: "page"`) |
| A row that names a product group and navigates somewhere | `leaf` (`link.type: "category"`) |
| ALL-CAPS header above a sub-list | usually `section` |

Input adapters you'll build (implementation is yours):
- **Figma**: read frame/layer/text node names + hierarchy.
- **Screenshot(s)**: vision/OCR to read tabs, headers, item labels, and indentation/grouping.
- **Website URL**: fetch and parse the nav (top tabs, dropdown sections, leaf links + their collection handles from hrefs).

---

## 10. Reference code you can build on

### 10.1 Types + factory helpers

```typescript
type MenuLinkType =
  | "category" | "page" | "empty" | "product"
  | "nativeWebPage" | "brands" | "tabBarId" | "";

interface MenuLink {
  type: MenuLinkType;
  payload: string;
  readableTitle?: string;
}

interface MenuItem {
  image: string;
  title: string;
  link: MenuLink;
  children: MenuItem[];
}

// Level-3 leaf that links to a Shopify collection.
function leafCategory(title: string, collectionId: string, readableTitle: string): MenuItem {
  return {
    image: "",
    title,
    link: { type: "category", payload: collectionId, readableTitle },
    children: [],
  };
}

// A grouping row inside a tab that only expands to reveal children.
function sectionHeader(title: string, children: MenuItem[]): MenuItem {
  return {
    image: "",
    title,
    link: { type: "page", payload: "", readableTitle: "" },
    children,
  };
}

// A top-level tab that holds children and has no direct destination.
function topTab(title: string, children: MenuItem[]): MenuItem {
  return {
    image: "",
    title,
    link: { type: "category", payload: "" },
    children,
  };
}

// A tab OR mid item that itself links directly to a collection.
function directCategory(title: string, collectionId: string, readableTitle: string, children: MenuItem[] = []): MenuItem {
  return {
    image: "",
    title,
    link: { type: "category", payload: collectionId, readableTitle },
    children,
  };
}
```

### 10.2 Build the tree → JSON

```typescript
interface ResolvedNode {
  title: string;
  kind: "tab" | "section" | "leaf";
  collectionId?: string;   // set for resolved category links
  readableTitle?: string;
  children: ResolvedNode[];
}

function toMenuItem(node: ResolvedNode): MenuItem {
  if (node.kind === "leaf") {
    return leafCategory(node.title, node.collectionId ?? "", node.readableTitle ?? node.title);
  }
  const children = node.children.map(toMenuItem);
  if (node.kind === "section") return sectionHeader(node.title, children);
  // tab: if it resolved to its own collection, link it; else empty container
  return node.collectionId
    ? directCategory(node.title, node.collectionId, node.readableTitle ?? node.title, children)
    : topTab(node.title, children);
}

function toMenuArrayJson(resolved: ResolvedNode[]): string {
  return JSON.stringify(resolved.map(toMenuItem), null, 2);
}

function toFullDocumentJson(resolved: ResolvedNode[], storekey?: string): string {
  const doc: any = { menu: resolved.map(toMenuItem), timestamp: Date.now() };
  if (storekey) doc.storekey = storekey;
  return JSON.stringify(doc, null, 2);
}
```

### 10.3 Category matcher

```typescript
interface VennCategory { id: string; name: string; handle: string; }

function findCategory(label: string, categories: VennCategory[], handleHint?: string): VennCategory | undefined {
  const norm = (s: string) => s.trim().toLowerCase().replace(/\s+/g, " ");
  const n = norm(label);
  return (
    categories.find((c) => norm(c.name) === n) ??
    categories.find((c) => norm(c.name).includes(n) || n.includes(norm(c.name))) ??
    (handleHint ? categories.find((c) => c.handle === handleHint) : undefined) ??
    categories.find((c) => c.handle === n.replace(/\s+/g, "-"))
  );
}
```

### 10.4 Validate before output

```typescript
function validateMenuItem(item: MenuItem, depth = 1): string[] {
  const e: string[] = [];
  const allowed = ["category","page","empty","product","nativeWebPage","brands","tabBarId",""];
  if (depth > 3) e.push(`Max depth exceeded at "${item.title}"`);
  if (typeof item.image !== "string") e.push(`image must be string at "${item.title}"`);
  if (typeof item.title !== "string") e.push(`title must be string at "${item.title}"`);
  if (!item.link || !allowed.includes(item.link.type)) e.push(`bad link.type at "${item.title}"`);
  if (typeof item.link?.payload !== "string") e.push(`link.payload must be string at "${item.title}"`);
  if (!Array.isArray(item.children)) e.push(`children must be array at "${item.title}"`);
  if (depth === 3 && item.children?.length) e.push(`level-3 must have empty children at "${item.title}"`);
  for (const c of item.children ?? []) e.push(...validateMenuItem(c, depth + 1));
  return e;
}
```

---

## 11. Product spec — inputs & outputs

### Inputs

| Input | Required | Notes |
|-------|----------|-------|
| `storekey` | Yes | Target store UUID (for collection resolution) |
| `environment` | Yes | `"uat"` or `"production"` |
| Design source | One of | Figma export, image upload(s), or website URL |
| `existingMenu` | Optional | To merge/append rather than replace |
| `genderPrefix` / hints | Optional | e.g. prefer "Women's …" collections when building the WOMEN tab |

### Outputs

1. **`menu` JSON** — paste-ready `MenuItem[]` (default), plus optional full-document variant.
2. **Warnings** — unresolved labels, ambiguous matches, depth violations, guessed matches.
3. **Mapping table** (optional) — `menuTitle → collectionId, readableTitle, matchScore` so a human can review.

---

## 12. Anti-patterns (these break validation or the app)

| ❌ Don't | ✅ Do |
|---------|------|
| Put a collection **handle** in `payload` | Put the numeric **collection ID** |
| Nest deeper than 3 levels | Flatten or merge |
| Omit `"children": []` on a leaf | Always include it |
| Omit `"image"` | Set `"image": ""` |
| Emit Shopify menu shape (`handle`, `parentId`, `type:"collection_link"`) | Emit Venn `MenuItem` shape only |
| Emit `infoConfig.shopMenu` (processed) shape | Emit raw dashboard `menu` array |
| Reuse MEN collection IDs on the WOMEN tab | Re-resolve each tab's collections |
| Invent a `link.type` | Only use the §4 allow-list |

---

## 13. Test cases (verify your generator)

**TC1 — flat tab.** Labels: `NEW IN`, `BESTSELLERS`, `SALE` under one tab → 1 `topTab`, 3 `leafCategory`, each `children: []`.

**TC2 — section grouping.**
```
MEN
  CATEGORY
    HOODIES
    JOGGERS
```
→ tab → `sectionHeader("CATEGORY")` (`type: page`) → 2 `leafCategory`.

**TC3 — unresolved collection.** Label `MYSTERY COLLECTION` not in store → `payload: ""`, `readableTitle: "MYSTERY COLLECTION"`, warning emitted.

**TC4 — depth guard.** A 4-level design → validation error (or auto-flatten with a warning).

**TC5 — duplicate tab.** Build WOMEN from the MEN structure → every leaf re-resolved against women's collections; no men's IDs reused.

---

## 14. One-line goal

> From a **visual/website menu hierarchy + a storekey**, output a validated **`MenuItem[]` JSON array** (schema in §3–§5), with each collection link carrying the store's **numeric collection ID** in `payload` and its title in `readableTitle`, ready to paste into the Venn Store Editor's **"Shop Menus - Dashboard only"** document.
