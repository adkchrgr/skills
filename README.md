# skills

Claude skills, in the layout Cowork and Claude Code expect (`skills/<name>/SKILL.md`).

## Skills

| Skill | What it does |
|---|---|
| [`grocery-run`](skills/grocery-run/) | Multi-store grocery price comparison and cart building in Chrome, with hard dietary screening |

## grocery-run

Compares per-unit prices across Sam's Club, Instacart (Aldi / Market 32), Hannaford, Azure Standard, and Amazon; screens every product against hard dietary filters; fills carts and stops before checkout.

**Screening filters** — gluten (including cross-contamination advisories), seed oils, Taylor Farms co-packed produce, and cell-cultivated meat. Blocks are hard: nothing non-compliant enters a cart, and substitutions are always reported rather than made silently.

**What makes it fast.** The bottleneck in this kind of task isn't loading pages, it's *finding* products and pulling whole pages into context to read one number. The skill addresses that three ways:

- A cached product-ID file, so repeat runs navigate straight to a product instead of searching for it
- Per-store in-page JS extractors that return structured rows instead of page text
- A dated price history, so each run only re-checks what's plausibly stale

The per-store extraction table in `SKILL.md` is empirically derived, not assumed — including where it *doesn't* work. Sam's renders products client-side, so `__NEXT_DATA__` is useless; Hannaford blocks in-page JS entirely and its search box ignores Enter. Those findings are written down so the next run doesn't rediscover them.

**Ground rules.** Never fetches outside the authenticated browser session — every price sits behind a session cookie and several of these sites bot-block. Never solves a CAPTCHA. Never completes a purchase.

## Setup

The skill maintains three files in your working folder:

| File | Holds |
|---|---|
| `grocery-staples.md` | Recurring list, known-good products, rejections, store verdicts |
| `price-history.csv` | Dated observations: `date,item,store,brand_product,size,price,unit_price,unit,notes` |
| `product-ids.md` | Direct product URLs/IDs per store |

These are personal operating data and are deliberately **not** in this repo. The skill creates them on first run.

Store lists, dietary filters, and the retailers themselves are all worth editing to fit your own situation — the structure generalizes further than the specific stores do.
