# Attribution extraction per source

Every successfully resolved image gets `author`, `license`, and `source_url` written to the asset manifest. The renderer composes a credit line per source per the spec's §5.7. This file describes how to extract those fields.

## 1. Wikimedia (Commons via `rest-summary`, `wikidata-verified`, `wikivoyage-banner`, `commons-file-search`, `commons-category`, `override-commons`)

All Wikimedia-hosted images live on Commons. Fetch attribution via the imageinfo API:

```
https://commons.wikimedia.org/w/api.php?action=query&titles=File:<filename>&prop=imageinfo&iiprop=extmetadata&format=json
```

Substitute `<filename>` with the URL-encoded Commons filename (spaces as `%20` or `_`).

### Extraction

```bash
imageinfo_json=$(curl -sS -A "myfootmarks/0.0.1 (graner@gmail.com)" \
  "https://commons.wikimedia.org/w/api.php?action=query&titles=File:${filename_encoded}&prop=imageinfo&iiprop=extmetadata&format=json")

artist_html=$(echo "$imageinfo_json" | jq -r '.query.pages | to_entries[0].value.imageinfo[0].extmetadata.Artist.value // empty')
license=$(echo "$imageinfo_json" | jq -r '.query.pages | to_entries[0].value.imageinfo[0].extmetadata.LicenseShortName.value // "unknown"')

# Strip HTML from artist (extmetadata embeds <a> / <span>)
author=$(echo "$artist_html" | sed -E 's/<[^>]+>//g' | sed -E 's/&amp;/\&/g; s/&nbsp;/ /g' | tr -s ' ' | sed -E 's/^[[:space:]]+|[[:space:]]+$//g')

source_url="https://commons.wikimedia.org/wiki/File:${filename_encoded}"
```

If `Artist` is absent or empty, set `author: "unknown"`. If `LicenseShortName` is absent, set `license: "unknown"`. Do not fail the resolution — the image is usable with `unknown` attribution.

### Renderer credit line

`Photo: <author> / Wikimedia Commons, <license>`

## 2. Openverse (`openverse`)

Attribution comes from the Openverse result directly — no second HTTP call needed.

```bash
# Given openverse_result_json (a single result object from the API)
image_url=$(echo "$openverse_result_json" | jq -r '.url')
author=$(echo "$openverse_result_json" | jq -r '.creator // "unknown"')
license_code=$(echo "$openverse_result_json" | jq -r '.license // "unknown"')
license_version=$(echo "$openverse_result_json" | jq -r '.license_version // ""')
source_url=$(echo "$openverse_result_json" | jq -r '.foreign_landing_url')

# Combine license code + version into a human-readable string
case "$license_code" in
  cc0) license="CC0";;
  pdm) license="Public domain";;
  by) license="CC BY ${license_version}";;
  by-sa) license="CC BY-SA ${license_version}";;
  by-nc) license="CC BY-NC ${license_version}";;
  *) license="${license_code}";;
esac
```

### Renderer credit line

`Photo: <author> via Openverse, <license>`

## 3. og:image (`subpage-og-image`, `homepage-og-image`)

Venue-hosted og:image has no formal attribution surface. The renderer credit is the venue's domain only.

```bash
# Given og_image_url and the originating venue URL (e.g., https://belfry.bc.ca/shows/casey-and-diana)
domain=$(echo "$venue_url" | awk -F/ '{print $3}')

author="unknown"
license="all rights reserved (venue website)"
source_url="$venue_url"
```

### Renderer credit line

`Photo: <domain>` (e.g., `Photo: belfry.bc.ca`)

The "all rights reserved" license is the **default**. The trip-book's keepsake use is fair-use editorial reference at thumbnail scale; the license string is documentary, not legal advice. If a venue explicitly asks for image removal, the user can add an `image_overrides:` entry mapping that place to `"skip"`.

## 4. Override forms (`override-commons`, `override-url`, `override-skip`)

| Override form | author | license | source_url |
|---|---|---|---|
| Commons URL or raw filename (`override-commons`) | extracted via §1 | extracted via §1 | Commons file page URL |
| Arbitrary HTTPS URL (`override-url`) | `"unknown"` | `"user-provided"` | the override URL itself |
| `"skip"` (`override-skip`) | n/a | n/a | n/a (renders typography-only) |

## Edge cases

- **Author HTML is just whitespace** after stripping → set `author: "unknown"`.
- **Author contains a Wikimedia username with no real-name** (e.g., `User:DiliffMan`) → keep the username as-is; it's a valid attribution.
- **License is a non-CC string** (e.g., `GFDL`, `OGL`) → keep verbatim; the renderer prints it as given.
- **Multiple licenses listed** → take the first (extmetadata's `LicenseShortName` is typically the most permissive primary).
- **Commons API returns a redirect** (file moved) → follow once; don't loop.
