# gtm-persist-google-ads-url-query-strings

A **Google Tag Manager (GTM) container export** (JSON) — not a single tag template — that you import directly into a GTM workspace to persist five Google Ads–related URL query parameters as first-party cookies. It follows the exact same `Cookie Creator` / `Get Root Domain` write-once-then-renew pattern used across this author's other GTM container exports (the [social-media click-ID](https://github.com/drewspen/gtm-persist-various-social-media-marketing-url-query-strings) and [Meta/Facebook](https://github.com/drewspen/gtm-persist-meta-facebook-url-query-strings) containers), here scoped specifically to parameters Google's own ad products generate.

This README is based on a direct inspection of the container export's actual contents (15 tags, 15 triggers, 16 variables, 6 folders, 5 built-in variables, and 2 bundled custom templates), cross-checked against the repository's own description.

## The 5 parameters covered

| Parameter | What it identifies |
|---|---|
| `gclid` | Google Ads' click identifier — the primary click ID used for offline conversion imports and Enhanced Conversions. |
| `dclid` | Google Display & Video 360's (formerly DoubleClick's) click identifier. |
| `srsltid` | Google Merchant Center / Shopping listing click identifier. |
| `gad_campaignid` | The Google Ads campaign ID associated with the click — appended automatically by Google Ads' [auto-tagging](https://support.google.com/google-ads/answer/6260813) when final URL suffixes/valuetrack parameters are configured to include it. |
| `gad_source` | Identifies which Google surface/network served the ad (e.g., Search, Display, YouTube, Discover), also part of Google Ads' auto-tagging output. |

## How the tag/trigger logic works

Each of the five parameters uses the same **three-tag pattern** established across this author's containers, all firing on `INIT`-type triggers (evaluated via trigger filter conditions on page load):

1. **`Create <param>`** — session cookie (`_my_<param>`), unconditionally overwritten with the most recent value whenever the parameter is present in the URL — capturing the *current/most recent* touch for that browsing session.
2. **`Create 1st <param>`** — persistent cookie (`_my_1st_<param>`), written only once, the first time the parameter is seen and the cookie doesn't already exist — locking in first-touch attribution.
3. **`Rewrite 1st <param>`** — fires whenever the persistent cookie already exists, re-writing it with its own existing value purely to renew its 12-month expiration on each return visit, without ever changing the originally-captured value.

This is identical in mechanics to the pattern documented in this author's [UTM-persistence](https://github.com/drewspen/gtm-persist-utm-url-query-strings-1st-party-cookie), [referral/landing/fbclid](https://github.com/drewspen/gtm-persist-original-source-landing-fbclid), [social-media click-ID](https://github.com/drewspen/gtm-persist-various-social-media-marketing-url-query-strings), and [Meta/Facebook](https://github.com/drewspen/gtm-persist-meta-facebook-url-query-strings) containers.

## Why persist these specifically?

`gclid`, `dclid`, and `srsltid` are what Google's own offline conversion import and Enhanced Conversions features use to match a later conversion (e.g., a form submission, purchase, or signed-up lead) back to the specific ad click that drove it, independent of cookie-based tracking. `gad_campaignid` and `gad_source` add campaign- and traffic-source-level context on top of that — useful for reconciling which specific campaign or Google surface (Search vs. Display vs. YouTube, etc.) drove a conversion, without needing to cross-reference click IDs back in Google Ads after the fact.

## Cookie configuration

- **Cookie names:** `_my_<param>` (session) and `_my_1st_<param>` (persistent), e.g. `_my_gclid`, `_my_1st_gclid`, `_my_gad_campaignid`, `_my_1st_gad_campaignid`, etc.
- **Domain:** `{{Root Domain}}` (via the bundled **Get Root Domain** template) — shared across subdomains of the same root domain (the same cross-domain limitation noted in this author's other click-ID containers applies here too — this does not span genuinely separate top-level domains).
- **Path:** `/`
- **Expiration:** 12 months for all `_my_1st_*` cookies; session-only for the plain `_my_*` cookies.
- **SameSite:** `None`, with the SameSite checkbox enabled, on all 15 tags.
- **Secure:** disabled (`checkbox1Secure: false`) on all 15 tags — see [Known issue](#known-issue-samesitenone-without-secure) below.
- **Consent:** every tag requires **`analytics_storage`** consent (`consentStatus: NEEDED`), matching this author's referral/landing/fbclid, social-media click-ID, and Meta/Facebook containers.

## Custom templates

Both bundled templates are actively used by every tag/variable in this container:

| Template | Purpose |
|---|---|
| **Cookie Creator** | Sets each cookie via sandboxed JavaScript, avoiding the CSP `script-src`/SHA-256 hash allowances a raw Custom HTML–based cookie script would require. |
| **Get Root Domain** | Extracts the registrable root domain from `{{Page Hostname}}`, used as every cookie's `Domain` value (via the single `Root Domain` variable). |

## Variables

For **each** of the five parameters, three variables are defined (15 total), plus one shared variable:

| Variable pattern | Type | Purpose |
|---|---|---|
| `<param> URL Query Value` | URL (Query component) | Reads the raw incoming `<param>` value from the query string; feeds the `Create`/`Create 1st` tags. |
| `<param> 1st Cookie Value` | 1st-Party Cookie | Reads back the persistent `_my_1st_<param>` cookie; used by the `Create 1st`/`Rewrite 1st` trigger conditions and `Rewrite 1st`'s cookie value. |
| `<param> from Cookie` | 1st-Party Cookie | Reads back the session `_my_<param>` cookie — not consumed by any tag inside this container itself; included for use elsewhere (e.g., a lead form or Google Ads Enhanced Conversions integration). |
| `Root Domain` | Custom (Get Root Domain) | Root domain from `{{Page Hostname}}`, used as every cookie's `Domain`. |

Five GTM built-in variables are enabled (`Page URL`, `Page Hostname`, `Page Path`, `Referrer`, `Event`); only `Page Hostname` is directly referenced by this container's own logic (feeding `Root Domain`).

As with this author's other click-ID persistence containers, **no `Shared Event Settings` GA4 variable is included here**, despite the repository description mentioning these values "can be passed to the shared events settings variable, then configure an event or user custom definition in GA4." You're expected to build that GA4 wiring yourself using the `*from Cookie`/`*1st Cookie Value` variables already provided.

## Prerequisites

1. **A GTM web container** — ideally a scratch/sandbox workspace, since importing can create duplicate tags/triggers/variables if names collide with what's already in your container.
2. **Google Ads auto-tagging enabled**, and if you want `gad_campaignid`/`gad_source` specifically, confirm your account/campaigns are generating those parameters (they're part of Google's newer auto-tagging output, not present on every account configuration).
3. **A consent management setup** wired into GTM's consent mode capable of granting `analytics_storage` consent — all 15 tags require it to fire.
4. If you also plan to use this author's other cookie-persistence containers, note that this export bundles its **own copies of `Cookie Creator` and `Get Root Domain`** — importing more than one into the same GTM container may prompt you to resolve duplicate template conflicts.
5. If you intend to forward these values to GA4 or Google Ads' offline conversion/Enhanced Conversions imports, you'll need to **build that wiring yourself** — this container only handles capture and cookie persistence.

## Getting started

### Import into Google Tag Manager

1. In GTM, go to **Admin** → **Import Container**.
2. Choose `gtm-persist-google-ads-url-query-strings.json` from this repository.
3. Select the target container and **choose a new workspace** (recommended) rather than overwriting an existing one, so you can review the merge before publishing.
4. Choose **Merge**, and review the import summary — the export's account/container IDs (`999999999`, `GTM-AAAAAAAA`) are placeholder/scrubbed values, so GTM will import everything by name into your own container.
5. Confirm the import. Both bundled custom templates will import alongside the 15 tags, 15 triggers, and 16 variables that depend on them.

### Post-import checklist

- **Confirm your consent setup** grants `analytics_storage` appropriately, or none of these 15 tags will fire.
- **Review the `Secure` cookie setting** (see below) before publishing to a production HTTPS site.
- **Build the GA4/conversions-import side yourself** if you want these values reported downstream — create an Event Settings variable (or your own tag logic) referencing the relevant `*from Cookie`/`*1st Cookie Value` variables.
- **Test in Preview mode** with a URL containing one or more of these parameters (e.g., `?gclid=test123&gad_source=1`) and confirm the corresponding `_my_*` and `_my_1st_*` cookies are set with the expected values (using the correct lowercase `gad_campaignid` parameter name if testing that one), domain, path, and expiration; navigate to a second page and confirm the `_my_1st_*` cookies remain unchanged while their expiration silently renews.

### Using the persisted values

Once set, these cookies (and their corresponding read-back variables already included in this container) can be read back later in the visit — most commonly to populate hidden fields on a lead/contact form before submission to a CRM/MAP, giving you Google Ads campaign/click-level attribution for leads that convert several pages into the visit, and/or to forward `gclid` specifically to Google Ads' offline conversion import or Enhanced Conversions for improved server-side attribution.

## Known consideration: `SameSite=None` without `Secure`

As with this author's other GTM container exports, all 15 cookie tags here set `checkbox1SameSite: true` with `dropDownMenu1SameSite: "None"`, while `checkbox1Secure` is `false`. Modern browsers **require the `Secure` attribute whenever `SameSite=None` is used** — a cookie set this way will typically be **rejected outright**. You'll likely want to enable the Secure checkbox on all 15 tags after import so these cookies actually get set in current browsers.

## Notes

- The container is moderate in scope: 15 tags, 15 triggers, 16 variables, 6 folders, 5 built-in variables, and 2 custom templates (both used).
- This is the smallest of this author's ad-platform click-ID persistence containers reviewed so far (5 parameters, vs. 8 for the social-media container and 9 for the Meta/Facebook container).
- This is an unofficial, personal automation export and is not affiliated with or endorsed by Google. Always review a container export's tags, triggers, and variables — and test thoroughly in a sandbox workspace — before merging it into a production GTM container.

## License

No license file is currently included in this repository. Check with the repository owner before reusing this container export in a commercial or redistributed context.
