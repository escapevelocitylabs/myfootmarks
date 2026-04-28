# og:image heuristics

Homepage og:image is a logo / social-share noise-trap roughly 4 of 8 times across the prototype evaluation. Two layered filters keep the wrong things out:

1. **Filename reject filter** — applied to any og:image URL before accepting it.
2. **Link-text matching** for events — finds a sub-page whose link text fuzzy-matches the event name; preferred over guessing URL patterns.

## 1. Filename reject filter

Reject the og:image candidate if the **filename portion of its URL** (the basename, lowercased) contains ANY of the following tokens:

```
logo
share
social
placeholder
og-default
og_default
fb-
fb_
twitter-
twitter_
brand-
brand_
hero-default
default-
default_
favicon
icon
apple-touch
```

### Implementation

```bash
# Given og_url, extract filename and lowercase it
filename=$(basename "$og_url" | tr '[:upper:]' '[:lower:]')

# Reject patterns (extended-regex, anchored on substring)
reject_pattern="logo|share|social|placeholder|og-default|og_default|fb-|fb_|twitter-|twitter_|brand-|brand_|hero-default|default-|default_|favicon|^icon|apple-touch"

if echo "$filename" | grep -E -q "$reject_pattern"; then
  echo "rejected: filename matches reject filter"
fi
```

### Empirical evidence

In the v2 prototype evaluation across 18 entities and 7 sources, this filter:
- Rejected 4 of 4 logo og:images (true positives).
- Allowed all 4 venue-photo og:images through (zero false positives).

### When to update

If a real venue's photo og:image is rejected because of a token collision (e.g., a venue actually named "Default Coffee" with a filename like `default-coffee-storefront.jpg`), narrow the offending token (e.g., change `default-` to `default-photo` or anchor it more strictly). Always cite the false-positive case in the commit message.

## 2. Sub-page link-text matching (events archetype)

Event archetypes prefer sub-page og:images over homepage og:images. The link-text-matching algorithm:

1. Fetch the venue's homepage HTML using a browser-like UA + `Referer`.
2. Parse all `<a>` elements (href + visible text).
3. For each anchor, compute a **fuzzy match score** between its text and the event name. Strategy: case-insensitive longest-common-substring length divided by the longer of the two strings. Threshold: ≥ 0.55.
4. Among anchors above threshold, prefer those whose href contains an event-y path segment (`/show`, `/event`, `/performance`, `/calendar`, `/season`, `/whats-on`, `/program`).
5. Pick the highest-scoring anchor. Fetch its href as a sub-page. Extract the sub-page's `<meta property="og:image">`. Apply the filename reject filter from §1.
6. If no anchor scores above threshold, fall through to the next source in the stack — **do not guess URL patterns** like `/shows/{slug}` or `/events/{id}`.

### Implementation skeleton

```bash
# Pseudo-code; the resolver subagent implements this with grep/sed/jq + node -e
homepage_html=$(curl -sS -A "$BROWSER_UA" -H "Referer: https://www.google.com/" "$venue_url")

# Extract all <a> tags' href + text. node -e snippet:
node -e "
  const html = process.argv[1];
  const re = /<a[^>]+href=\"([^\"]+)\"[^>]*>([^<]+)<\/a>/g;
  const anchors = [];
  let m;
  while ((m = re.exec(html)) !== null) anchors.push({href: m[1], text: m[2].trim()});
  console.log(JSON.stringify(anchors));
" "$homepage_html"
```

(Real implementation may use a Python one-liner or a Bun script — `node` is the fastest path here since it's already available in this monorepo for prototype work, but the resolver brief stays language-agnostic. Use whichever is readily available in the subagent's tool surface.)

### Empirical evidence

In the v2 prototype, this strategy resolved:
- Belfry Theatre's "Casey and Diana" → `/shows/casey-and-diana` → sub-page og:image (a production still). True positive.
- Two festivals → festival sub-pages with editorial photography. True positives.
- One venue with no per-show pages → fall-through to the next source in the stack (commons-file-search hit). Correct fall-through.

### Failure modes (graceful)

| Failure | Behavior |
|---|---|
| Homepage fetch returns 403 / 404 / timeout | Skip this source; fall through. |
| Sub-page fetch returns 403 / 404 | Skip this source; fall through. |
| Sub-page has no og:image meta | Skip this source; fall through. |
| Sub-page og:image fails the filename reject filter | Skip this source; fall through. |
| No anchor above threshold | Skip this source; fall through. |

In every case, the resolver continues with the next source. og:image is **never** the only source attempted for an archetype; it is always a fallback.

## Confidence assignment

- `subpage-og-image` → `confidence: "medium"` (link-text matching narrows to the right sub-page; the og:image on a show-specific page is usually a production still).
- `homepage-og-image` → `confidence: "low"` ALWAYS, even after the filename reject filter passes. The audit table flags `low` confidence as a spot-check candidate; user can override if wrong.
