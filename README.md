# gtm_tracking_for_zid

Prompts and Claude skills for installing advertising-platform pixels on **Zid** stores through **Google Tag Manager**, built by an AI agent that has a GTM MCP server connected.

Zid pushes the same GA4-shaped dataLayer on every store, so each build is fixed. You give three inputs — GTM account, GTM container ID, pixel ID — and the agent audits the container, reuses what already exists, builds the tags in the workspace, verifies them, and stops. **Nothing here publishes a container.**

## What's inside

```
prompts/                              paste-and-run prompts (any agent with a GTM MCP server)
  meta-pixel-gtm-zid-prompt.md
  snap-pixel-gtm-zid-prompt.md
  tiktok-pixel-gtm-zid-prompt.md

skills/                               the same builds packaged as Claude skills
  meta-pixel-gtm-zid/     SKILL.md + reference.md
  snap-pixel-gtm-zid/     SKILL.md + reference.md
  tiktok-pixel-gtm-zid/   SKILL.md + reference.md
```

| Platform | Events built | Zid dataLayer events used |
|---|---|---|
| Meta Pixel | PageView, ViewContent, AddToCart, InitiateCheckout, AddPaymentInfo, Purchase | `view_item`, `add_to_cart`, `begin_checkout`, `add_payment_info`, `purchase` |
| Snap Pixel | PAGE_VIEW, VIEW_CONTENT, ADD_CART, START_CHECKOUT, ADD_BILLING, PURCHASE | same |
| TikTok Pixel | Base code (Pageview), ViewContent, AddToCart, InitiateCheckout, AddPaymentInfo, CompletePayment / Purchase | same |

## Requirements

- A GTM MCP server connected to the agent, with edit access to the client's container.
- The platform's tag template installed in the container from the Community Template Gallery: **Meta Pixel** (facebook), **Snap Pixel** (Snap Inc.), **TikTok Pixel** (tiktok). The agent stops and asks if it is missing.
- The pixel ID for the platform.

## Using a prompt

1. Open the prompt file and fill in the three lines under **INPUTS**.
2. Paste the whole file into the agent.
3. Read the report, test in GTM Preview, then publish yourself.

## Using a skill

Copy the skill folder into your Claude skills directory:

```bash
cp -r skills/meta-pixel-gtm-zid ~/.claude/skills/
```

Then ask for the work in plain language — for example *"set up the Meta Pixel through GTM on this Zid store"* — and give the three inputs when asked. `SKILL.md` holds the rules and the field notes; `reference.md` holds the Zid dataLayer reference and every GTM call and payload.

## Conventions

- **Naming.** Tags `EA - <Platform> - <event>`, platform variables `EA - <Platform> - <snake_case>`, variables shared between platforms `EA - Core - <snake_case>`. One folder per platform plus `EA - Core`. Change the `EA` prefix to your own if you fork this.
- **IDs live in Constant variables**, never hardcoded in a tag.
- **Existing triggers are reused**, never duplicated, and nothing that already exists in the container is renamed.
- **Advanced Matching differs per platform on purpose.** Meta receives the plain email and phone (its pixel hashes them); Snap and TikTok receive Zid's pre-hashed values.
- **Revenue is `ecommerce.value`** on every platform, so the platforms agree with each other.

## Notes

- All IDs, product data and customer values in these files are placeholders.
- Template IDs (`cvt_...`) are the values seen when these were written. Each build reads the real one from the installed template rather than trusting the number here.
- Platform behaviour changes. Check the report against the platform's own events tool before relying on the data.

## License

MIT — see [LICENSE](LICENSE).
