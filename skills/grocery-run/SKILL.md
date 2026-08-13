---
name: grocery-run
description: Shop for groceries in Chrome across Sam's Club, Instacart (Market 32 and Aldi), Hannaford, Azure Standard, and Amazon — comparing per-item unit prices, screening every product for gluten and for blocked sources (Taylor Farms, lab-grown meat, seed oils), and filling carts up to but never through checkout. Use this whenever the user mentions a grocery run, grocery list, shopping list, restocking food, ordering groceries, comparing grocery prices, an Azure drop or bulk pantry order, "what's cheapest at", or names any of these stores in a food-buying context — including when they just paste a list of food items with no other instruction.
---

# Grocery Run

You are shopping for someone with real dietary constraints and real money on the line. Two things matter more than speed: never putting a non-compliant item in the cart, and never spending their money without them. Everything else is optimization.

## The five stores

| Store | How to reach it | Notes |
|---|---|---|
| Sam's Club | `samsclub.com` | Membership pricing. Bulk sizes — always compute unit price. Items split between "Delivery/Shipping" and "Pickup at club"; availability differs by fulfillment mode. |
| Market 32 | Instacart storefront (`instacart.com/store/market-32/...`) | Instacart prices are often marked up above shelf price. Note this when it looks material. |
| Aldi | Instacart storefront (`instacart.com/store/aldi/...`) | Mostly private label. Instacart publishes **no ingredient text** for Aldi private label — front-of-pack claims in the product image are often all you get. |
| Hannaford | `hannaford.com` | Own ordering system. Requires a selected store for accurate pricing. Throws bot-verification sliders on automated browsing. |
| Azure Standard | `azurestandard.com` | Bulk natural-foods buying club. Runs on a **monthly cycle**, not same-week — see below. |

Amazon is worth a spot-check on shelf-stable brand-name items, but in practice it rarely wins against a club store. Don't route a whole basket through it.

Storefront slugs and URL shapes drift. Treat the table as a starting guess: navigate to the site's own search box if a constructed URL 404s.

### Azure Standard works on a different clock

Azure is not a substitute for the others and shouldn't be compared head-to-head on a weekly list. It delivers to a designated **drop point** on a monthly truck route, with an order deadline several days ahead of the drop date. There's an order minimum, and most drops add a percentage shipping fee — find it and apply it to every Azure price before comparing.

Practically:

- **Check the cycle first.** Find the drop point, the next drop date, and the cutoff before adding anything. If the cutoff has passed, say so rather than silently building a cart they can't submit.
- **Azure's edge is narrower than it looks.** In practice it wins on **bulk cooking fats and oils** — coconut, olive, avocado — often by 2-3x. It frequently *loses* on canned fish, tallow, flour blends, nut butters, and coffee, where a club store or discount grocer is cheaper. Verify per item; don't assume the natural-foods retailer is cheaper because it's bulk.
- **Keep a running "next drop" list.** When a staple is cheaper in bulk but isn't needed this week, don't force it into the weekly cart.
- **Screening is easier here but not automatic.** The catalog skews organic and single-ingredient, and product pages usually show full panels. Still run every filter — "natural foods retailer" is not the same as "no soybean oil."

The natural rhythm is a weekly run across the grocery stores plus a monthly Azure order timed to the drop. If the user hasn't set that up, offer it.

## Getting set up

Use the Chrome MCP tools (`mcp__claude-in-chrome__*`), not computer-use — these are web apps, and DOM-aware tools are far faster and less error-prone than clicking pixels. Load what you need in one `ToolSearch` call:

```
select:mcp__claude-in-chrome__tabs_context_mcp,mcp__claude-in-chrome__navigate,mcp__claude-in-chrome__computer,mcp__claude-in-chrome__read_page,mcp__claude-in-chrome__get_page_text,mcp__claude-in-chrome__find,mcp__claude-in-chrome__form_input,mcp__claude-in-chrome__javascript_tool,mcp__claude-in-chrome__tabs_create_mcp,mcp__claude-in-chrome__browser_batch
```

`javascript_tool` is the workhorse — see the price-gathering section. Load it up front.

Open one tab per store and keep them open for the whole run. You will bounce between them constantly while comparing.

**Login and store selection.** Check each tab's logged-in state before shopping. If a store needs a login, a ZIP code, a delivery address, or a store selection, stop and ask the user to handle it — don't guess at addresses or attempt credentials. Confirm the Hannaford store, the Instacart delivery address, and the Azure drop point match where they actually want delivery. A store left set to the wrong region produces confident, completely wrong numbers — check it every run, not just the first.

**Never fetch these sites outside the browser.** curl, wget, requests, and friends all fail here: every price sits behind a session cookie, and several of these sites bot-block aggressively. Work inside the authenticated browser session or not at all.

## The three files this skill lives on

Read all three at the start of a run. They carry knowledge between runs and are what make later runs fast.

| File | Holds | Written when |
|---|---|---|
| `grocery-staples.md` | Recurring list, known-good products, rejections, store verdicts | End of every run |
| `price-history.csv` | Dated price observations, one row per item per store | Every time a price is read |
| `product-ids.md` | Direct product URLs/IDs per store, plus the URL shapes | Whenever a new product is resolved |

If they don't exist, offer to create them from this run — the first run pays for all the later ones.

## Gathering prices without burning the whole session

The slow part of a grocery run is not loading pages — it's *finding* products, and hauling whole pages into context to read one number. Three habits fix that.

**1. Go straight to cached product IDs.** `product-ids.md` stores the direct URL for each item at each store. Navigating to a known ID is one step; searching for it is three or four. Whenever you resolve a new product, write its ID to that file.

**2. Extract with in-page JS instead of reading whole pages.** Pull only the fields you need. On a Sam's search page this returns every result as a structured row in one call:

```js
var out=[],seen={};
document.querySelectorAll('a[href*="/ip/"]').forEach(function(a){
  var m=a.getAttribute('href').match(/\/ip\/[^/]+\/(\d+)/); if(!m) return;
  var id=m[1]; if(seen[id]) return;
  var card=a.closest('div[class*="sc-"], li, article') || a.parentElement.parentElement;
  var t=(card?card.innerText:a.innerText).replace(/\n+/g,' ');
  var name=(a.innerText||'').replace(/\n+/g,' ').trim(); if(!name) return;
  var price=(t.match(/\$\d+\.\d{2}/)||[''])[0];
  var unit=(t.match(/\$[\d.]+\/(?:oz|lb|ea|foz|ct)/)||[''])[0];
  if(!price) return;
  seen[id]=1; out.push({id:id,name:name.slice(0,60),price:price,unit:unit});
});
JSON.stringify(out.slice(0,10))
```

Same idea for ingredient panels — one line back instead of a whole page:

```js
document.body.innerText.split('\n')
  .filter(l => /^\s*(Ingredients:|Contains:)/i.test(l)).join(' || ')
```

Tested method per store:

| Store | How to read prices |
|---|---|
| Sam's Club | JS DOM extractor above. Products render client-side — `__NEXT_DATA__` holds only page config, so don't parse it. Product pages **do** carry full ingredient panels. |
| Instacart | JS over `button[href*="/products/"]` to harvest IDs; `get_page_text` for the price list, which comes back clean. Ingredient accordions are empty for Aldi private label — fall back to zooming the package image for front-of-pack claims. |
| Hannaford | `get_page_text` only — `javascript_tool` is blocked on this domain. Output is already well structured (`$0.91 \| 15 oz can \| $0.97 /per pound`). Search via `hannaford.com/product-search/<query>`; the header search box needs `form_input` plus a click on the magnifier, as Enter doesn't submit. |
| Azure Standard | Navigate `/shop/search/<query>`, then JS. Results carry a unit-price range across sizes, so bulk tiers are visible without opening each product. |
| Amazon | JS over `[data-component-type="s-search-result"]`, reading `h2` and `.a-price .a-offscreen`. |

**3. Batch everything.** `browser_batch` chains navigate → wait → extract for several products in one round trip. Four or five products per call is comfortable. Sequential single calls are the main reason a run drags.

## Only re-check what's likely to have moved

`price-history.csv` holds observations as `date,item,store,brand_product,size,price,unit_price,unit,notes`. Read it first and re-price only what's stale:

- Fresh produce and meat — weekly
- Dairy and eggs — monthly
- Canned, dry, and shelf-stable goods — quarterly
- Anything flagged as a sale price — always, since sales expire

Everything else carries forward. This is the difference between a 15-call run and an 80-call one.

Append rather than overwrite, so trends stay visible. When a unit price moves more than ~10% from the last observation, call it out — that's either a sale worth stocking up on or a quiet increase worth re-shopping.

For anything beyond simple comparison — ranking a whole basket, normalizing unit prices across mixed sizes — write the rows to a file and let a script do the arithmetic. Mental math over dozens of unit prices is where errors creep in.

**Watch for bulk tiers.** Club stores list #10 cans and case packs alongside retail sizes, and the unit price can beat the discount grocer outright. The extractor surfaces these automatically; eyeballing search results misses them. Flag the storage reality too — a 106 oz can of corn is only cheaper if it actually gets eaten.

## Building the list

Two sources, and you need both:

1. **The staples file** — `grocery-staples.md`.
2. **What they say in chat** — one-offs, swaps, "skip the eggs this week."

Read the staples file first, then reconcile. If chat contradicts the file, chat wins for this run; ask whether they want the file updated permanently.

The staples file carries accumulated knowledge, and that's most of its value:

```markdown
## Weekly
- Eggs, 18ct
- Bananas

## Azure next drop
- Drop point: [name], next drop [date], cutoff [date], +[X]% shipping fee
- Coconut oil, 1 gal

## Known-good products
- Rao's Marinara (GF, olive oil) — Hannaford $8.99, Sam's 2-pk better unit price

## Rejected
- [brand] chicken broth — barley malt
- [brand] mayonnaise — soybean oil
```

A run that adds nothing to this file has thrown away most of its work.

## Screening — the part that must not be rushed

Every item goes through screening **before** it enters a cart. An item added and later removed is an item that might survive to checkout.

### Gluten — hard block

Block: wheat (incl. spelt, farro, durum, semolina, einkorn, kamut, wheat starch), barley, malt / malt extract / malt vinegar / malt flavoring, rye, triticale, brewer's yeast, oats **unless** labeled certified gluten-free.

Also block "Contains: Wheat" and "may contain wheat" / "processed on shared equipment" advisories. Cross-contamination advisories are a real risk, not boilerplate.

Ambiguous ingredients — do not guess:
- Modified food starch, dextrin, natural flavors, "spices," maltodextrin, caramel color, soy sauce
- In the US these are usually corn- or soy-derived and fine, but not always. If the package carries a certified-GF mark or a clear "gluten free" claim, accept it. Otherwise treat it as unresolved and surface it rather than silently including or excluding it.

Prefer third-party certification (GFCO, NSF) over an unverified front-of-pack claim.

**Don't infer a block from the absence of a label.** A missing "gluten-free" tag in a retailer's UI is weak evidence — plenty of compliant products carry the claim only on the package. Zoom the product image before flagging. Guessing wrong in the cautious direction still wastes the user's time and erodes trust in the list.

### Taylor Farms — hard block

Hardest of the three to catch, because Taylor Farms co-packs private-label salad kits, chopped salads, veggie trays, and stir-fry blends under retailers' own brands. The front label will not tell you.

Where to look: the product image's back panel for "Distributed by Taylor Farms" or "Taylor Farms Retail, Inc.", the Salinas CA / Yuma AZ address, or an establishment code. Also check the detail page's brand/manufacturer fields.

When you cannot determine the packer for a bagged salad or fresh-cut vegetable — which will be often — **do not add it**. Say so plainly. Whole, uncut produce sidesteps the problem entirely and is usually cheaper, so reach for it first.

### Lab-grown / cultivated meat — hard block

Block anything labeled cell-cultivated, cell-cultured, cultivated, or lab-grown, and known producers: UPSIDE Foods, GOOD Meat / Eat Just, Wildtype, Mission Barns, Believer Meats, Aleph Farms.

Availability in US grocery retail is currently near zero, so expect this filter never to fire. Don't hunt for it — just recognize the labels. Distinct from plant-based substitutes; if one comes up, ask rather than assume.

### Seed oils — hard block

Block: canola / rapeseed, soybean, corn, sunflower, safflower, cottonseed, grapeseed, rice bran, and unqualified "vegetable oil" (nearly always soybean or a soy/canola blend).

Accept: olive, avocado, coconut, butter/ghee, tallow, lard, palm, and unrefined nut oils.

Three practical notes. High-oleic sunflower and safflower are chemically distinct and some avoiders accept them — flag rather than auto-block, ask once, record the answer. **Sunflower lecithin is an emulsifier, not an oil**, and appears in trace amounts in chocolate coatings — flag, don't block. And this filter is the one most likely to wipe out an entire category (chips, crackers, mayo, dressings, most packaged snacks). When it does, say so rather than quietly returning nothing.

### When something is blocked

Never substitute silently. Find one or two compliant alternatives, add the best one, and report what you swapped and why. If nothing compliant exists anywhere, leave it off and say so — a clear gap they can solve beats a cart with something in it they can't eat.

## Comparing and buying

The goal is **cheapest per item across the stores**, splitting the order where that wins. Compare on **unit price** (per oz, per lb, per count), never shelf price — that's what makes a club bulk pack commensurable with a single unit at a discount grocer.

Weigh against unit price:
- **Timing.** A cheaper Azure price on something needed this week isn't actually cheaper. Buy it now locally and add the bulk version to the next-drop list.
- **Instacart markup.** Listed prices often sit above shelf price. Use them as-is — it's what the user pays — but mention notable inflation.
- **Bulk that won't get eaten.** A 5 lb bag of spinach is not cheaper than 1 lb if 4 lb rot. Bulk logic works for shelf-stable goods; apply judgment to perishables.
- **Fees and minimums.** An extra store for one $3 item usually loses once delivery fees and minimums are counted. Consolidate small remainders into a store already in use.
- **Shipping vs. pickup at the same retailer.** Check whether a shipped item is stocked at the club before letting it ship — paying shipping on the same day as a pickup order at that club is pure waste.

For a large list, work store by store rather than researching everything first. Fewer tab switches, and partial progress survives interruption.

### Stopping at checkout — absolute

Fill the carts. Then stop. Do not enter payment details, do not place an order, do not click "Place order," "Submit," or "Checkout," and do not confirm a delivery slot that finalizes an order.

Hand back the cart URLs and let the user do the final review and purchase. It's their money, and the cart is the last place they can catch a screening mistake.

## What to report at the end

Close with a short summary they can act on, not a transcript of your browsing:

```markdown
## Grocery run — [date]

**Carts ready** (review and checkout yourself):
- Sam's Club — 6 items, ~$X → [cart link]
- Hannaford — 11 items, ~$Y → [cart link]
- Instacart / Aldi — 4 items, ~$Z → [cart link]
- Azure Standard — 3 items, ~$W → [cart link] — **cutoff [date]** for the [date] drop
Estimated total: $XXX

**Swapped**
- [item] → [replacement] (original had soybean oil)

**Couldn't source compliant**
- [item] — every option at all stores contained barley malt

**Needs your call**
- [item] — bagged salad, packer not shown; skipped

**Price moves since last run**
- [item] down 20% at [store] — stock up
```

Then write back to all three files — new prices to `price-history.csv`, newly resolved products to `product-ids.md`, verdicts and rejections to `grocery-staples.md`. Offer to schedule the run if it looks like a weekly habit.

## Handling friction

**Client-rendered pages.** All of these sites render with JavaScript. Read after the page settles — an empty or nav-only result means it hasn't rendered, not that the item doesn't exist.

**Bot walls and CAPTCHAs.** Stop and ask the user to clear it. Never attempt to solve or work around one.

**Search boxes that don't submit.** Several of these sites ignore a typed Enter. `form_input` to set the value, then click the submit icon.

**Item not found.** Try a shorter query — site search is usually literal, so "gf tamari" fails where "tamari" works. Then a category browse, then move on. Note it as unsourced and keep going.

**Out of stock.** Check the other stores before declaring a gap. If it's out everywhere, ask before substituting — substitution preferences are personal and you'll usually guess wrong.
