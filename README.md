# Cub-Crafters

Daily aggregator of CubCrafters aircraft classified listings (Carbon Cub, Top
Cub, XCub, FX, NX, etc.) from [Barnstormers.com](https://www.barnstormers.com),
published as a static page (`docs/index.html`) meant to be embedded via
`<iframe>` on taildraggers.com.

Controller.com was evaluated (in the companion [Aeronca](https://github.com/taildraggers/aeronca)
repo) and dropped: its search results are only reachable through an internal
client-side widget (not a plain URL), which a headless browser can't drive
reliably for an unattended daily job.

Note: in the companion [Aviat](https://github.com/taildraggers/aviat) repo,
Barnstormers' single-manufacturer category page (their "Aviat Aircraft" hub)
turned out to include unrelated aircraft — including CubCrafters models —
mixed in with no distinguishing HTML markup. The same happened here in
reverse: real runs found a Piper Super Cub, a Comp Air, a Cessna 150 raffle,
a Sonex raffle, and an Aeronca 7EC raffle mixed into the "CubCrafters"
category alongside genuine listings. `scraper/barnstormers.py` now filters
by title against a small allowlist of CubCrafters product names
(`TARGET_MODEL_PHRASES`: CubCrafters, Carbon Cub, XCub, CC11/CC19, Top Cub)
before publishing, matching the approach used in the Aviat repo.

## How it works

- `scraper/barnstormers.py` searches Barnstormers.com's CubCrafters category for
  listings, follows pagination, then visits each listing's detail page to pull
  out the price, location, and posted date (falling back to regex heuristics
  over the visible text since the site doesn't expose structured data). The
  title is derived from the listing URL's own SEO slug, since every detail page
  shares one generic `<title>`/`<h1>`.
- On top of that brand allowlist, only whole-aircraft-for-sale listings are kept.
  Each ad's title must match a recognized CubCrafters model (Carbon Cub SS/EX/UL,
  XCub, CC11-160, FX, Top Cub, etc. - see `_MODEL_RULES` in
  `scraper/barnstormers.py`); titles that read as parts, accessories, services,
  or raffles are dropped. Every surviving listing's title is rewritten to a
  canonical **`YEAR CubCrafters MODEL`** form when the ad states a model year
  (e.g. `2020 CubCrafters XCub`), or just **`CubCrafters MODEL`** when it doesn't
  - a missing year isn't disqualifying, since plenty of genuine ads simply don't
  state one in the title - regardless of how the original ad was worded, so the
  page reads consistently.
- `main.py` runs the scraper, de-duplicates results, and renders them into
  `docs/index.html` titled **"Other CubCrafters Ads on the Web"**, with
  one row per listing: Title (linked to the original ad), Price, Location,
  Date Posted, and Site Posted On. Links use `rel="noopener noreferrer"` and
  the page sets a `no-referrer` meta policy, so Barnstormers never sees that
  the click came from taildraggers.com.
- `.github/workflows/daily-scrape.yml` runs the whole thing once a day (13:00 UTC),
  commits the regenerated `docs/index.html` if it changed, and can also be triggered
  manually from the Actions tab (`workflow_dispatch`).

## One-time setup: enable GitHub Pages

This repo publishes `docs/index.html` as a plain static file — GitHub Pages just needs
to be pointed at it once:

1. Go to **Settings → Pages** in this repository.
2. Under **Build and deployment → Source**, choose **Deploy from a branch**.
3. Branch: `main`, folder: `/docs`. Save.
4. GitHub will publish the page at `https://taildraggers.github.io/cub-crafters/`
   (may take a minute or two the first time).

Also check **Settings → Actions → General**:
- **Actions permissions**: "Allow all actions and reusable workflows".
- **Workflow permissions**: "Read and write permissions" (needed so the daily
  job can commit the regenerated page back to the repo).

## Embedding on taildraggers.com

```html
<iframe
  src="https://taildraggers.github.io/cub-crafters/"
  title="Other CubCrafters Ads on the Web"
  style="width: 100%; height: 800px; border: 0;"
  loading="lazy">
</iframe>
```

## Running locally

```bash
pip install -r requirements.txt
playwright install --with-deps chromium
python main.py
```

This writes/overwrites `docs/index.html`.

## Notes

- If Barnstormers changes its markup or is briefly unreachable, the run logs will
  show a `[warn]`/`[error]` line pointing at what broke rather than failing silently.
- The scraper identifies itself with a browser-like `User-Agent` and adds a short
  delay between requests to be polite to the site.
- Only one Barnstormers category is currently configured
  (`category-17975-CubCrafters.html`). If listings turn out to be split across
  additional categories, add more URLs to `CATEGORY_URLS` in
  `scraper/barnstormers.py`.
