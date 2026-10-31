---
name: goofish
description: Goofish (Xianyu, 闲鱼) second-hand marketplace data through the goofish MCP tools. Use when the user wants Goofish or Xianyu listings searched, used-item prices in China compared, a listing's full details, a seller's profile, listings or reviews, listing comments or buyer Q&A, or listings that match a photo.
---

# Goofish (Xianyu, 闲鱼)

Five Goofish tools plus Apify's run and dataset tools, served by the `goofish` MCP server. Each Goofish tool starts a paid run on the user's Apify account and returns run metadata. The rows arrive later in a dataset. Every request follows the **run flow** below.

| Tool (client prefixes vary) | Use for |
|---|---|
| `zen-studio--goofish-xianyu-search-scraper` | Keyword search: listings with price, condition, location, seller stats |
| `zen-studio--goofish-xianyu-item-detail-scraper` | Full details for known item URLs or IDs |
| `zen-studio--goofish-xianyu-seller-scraper` | Seller profile, their listings, their reviews |
| `zen-studio--goofish-xianyu-visual-search-scraper` | Listings that look like a given image |
| `zen-studio--goofish-xianyu-item-comments-scraper` | Public comment thread and buyer Q&A (问大家) of listings |
| `get-actor-run` | Wait for and check a run |
| `get-dataset-items` | Read the rows |
| `get-dataset-list` | Find the separate `item-questions` dataset |
| `fetch-actor-details` | Full input reference and README of any of the five |

If only Apify's generic `call-actor` is available, call it with `actor: "zen-studio/<name>"` (for example `zen-studio/goofish-xianyu-search-scraper`) and the same input; the run flow is identical.

## Run flow

A tool call returns after at most 45 seconds, whether or not the run has finished. Many Goofish runs take longer, so treat the first response as a receipt, not the result.

1. **Start.** Call the Goofish tool with the recipe input and `"waitSecs": 45`. Keep the `runId` from the response. Done when you hold a `runId`.
2. **Poll.** Read `status`. While it is `READY` or `RUNNING`, call `get-actor-run` with `runId` and `waitSecs: 45`, and tell the user the run is still going between calls. Done when `status` is terminal: `SUCCEEDED`, `FAILED`, `ABORTED` or `TIMED-OUT`. A response that is already `SUCCEEDED` skips straight to step 3.
3. **Fetch.** Call `get-dataset-items` with `datasetId` from `storages.datasets.default.id` (the response's `nextStep` also names it). Pass `limit` (20 to 50) and a `fields` projection from **Reading results**; rows can carry 150+ fields. Page with `offset` until you have what the question needs. Done when every row the answer depends on is read.
4. **Answer.** Summarise for the user: prices in CNY (¥), item links (`url`), and anything filtered out.

One run per request. A run that is still `RUNNING` keeps working and billing; starting a second run for the same question pays twice. Resume polling the same `runId` instead, including after a client timeout or a new turn.

Terminal states other than `SUCCEEDED`: `TIMED-OUT` and `ABORTED` can hold partial rows, so fetch them and say the result is partial. `FAILED`: report `statusMessage` and change the input before retrying. A `SUCCEEDED` run whose item count reads 0 can lag for a few seconds; fetch once before concluding it is empty.

## Input recipes

Start small (`maxItems` 20 to 50) unless the user asks for more; every returned listing is billed.

**Search** (`goofish-xianyu-search-scraper`):

```json
{ "keyword": "任天堂 Switch OLED", "maxItems": 30, "sortBy": "priceAsc",
  "priceMin": 800, "priceMax": 1500, "regionIds": ["310000"],
  "quickFilters": ["filterPersonal"], "detailLevel": "full" }
```

- `keyword`: one term; `keywords`: a list searched under one shared `maxItems`. `searchUrls` (`https://www.goofish.com/search?q=...`) override both.
- Write keywords the way sellers write titles: Chinese for Chinese brands and categories (`小米`, `乐高`, `手机`). Pinyin and Latin spellings of Chinese brands return nothing (`xiaomi`, `huananzhi` fail; `小米`, `华南金牌` work). International product names such as `iPhone 15 Pro` work as-is.
- `sortBy`: `relevance`, `newest` (default), `priceAsc`, `priceDesc`, `priceDrop`, `modified`, `credit`.
- `priceMin` / `priceMax`: whole yuan, integers only. Sellers tag cheap accessories and other models with popular product names, so the low end of a range fills with them under `priceAsc`. Set `priceMin` near the item's realistic used price and check titles before answering.
- `regionIds`: 6-digit Chinese division codes. Province ends in `0000` (`310000` Shanghai, `440000` Guangdong), city ends in `00` (`440300` Shenzhen, `330100` Hangzhou). The tool schema lists every accepted code. Empty means nationwide.
- `publishedWithinDays`: `""`, `"1"`, `"3"`, `"7"`, `"14"` (strings). Add `"verifiedRecentOnly": true` to drop older listings labelled as recent; slower, fewer rows.
- `quickFilters`: `filterPersonal` (individual sellers), `filterAppraise` (inspection verified), `filterFreePostage`, `filterNew` (brand new), `inspectedPhone`, `filterHighLevelYxpSeller`, `gameAccountInsurance`, `filterOneKeyResell`.
- `detailLevel`: `full` (default) adds description, all images, condition and seller stats; `summary` returns id, title, price, URL, post time, location, seller name and thumbnail, much faster. Use `summary` for price scans, `full` when the user needs descriptions or seller quality. `"includeItemRecord": true` adds exact province/district and condition grade in `full` mode.
- One search shows about 1,500 results; larger `maxItems` (up to 20,000) keeps going across price ranges.

**Item details** (`goofish-xianyu-item-detail-scraper`):

```json
{ "startUrls": ["https://www.goofish.com/item?id=1045244773159", "947872178004"] }
```

Item URLs, bare numeric IDs or `m.tb.cn` share links; up to 1,000 per run. Invalid inputs return unbilled rows with `rowStatus: "invalid"`.

**Seller** (`goofish-xianyu-seller-scraper`):

```json
{ "sellerInputs": ["https://www.goofish.com/personal?userId=2215160932230"],
  "includeListings": true, "maxListings": 20, "includeReviews": true, "maxReviews": 50 }
```

- `sellerInputs`: profile URL, any item URL of that seller, share link, or numeric seller ID. Seller names and search URLs are not accepted; search first, then pass `seller.url` from a listing.
- `maxListings` default 10, `maxReviews` default 100; `0` means everything available.
- Listing filters (`priceMin`, `priceMax`, `publishedWithinDays` 1 to 3650, `minCondition` 1 to 10, `freeShippingOnly`) need `deepListings: true` (the default).
- Rows: one `rowType: "seller"` row per seller, then `rowType: "listing"` rows carrying `sellerId`. Reviews sit in the seller row's `reviews[]`.
- `listingDetails: true` adds full detail to each listing row and bills each such listing as two listing events. Ask before enabling on large sellers.

**Image search** (`goofish-xianyu-visual-search-scraper`):

```json
{ "imageUrls": ["https://example.com/photo.jpg"], "maxItems": 20, "picKeyword": "Nike" }
```

- `imageUrls`: public image links. For an image you only hold as bytes, pass it base64-encoded in `imageBase64`.
- `picKeyword` narrows matches by brand or model. Price, region, sort and quick filters work as in Search. `maxItems` up to 1,500.
- A clear photo of a single product matches best. Rows carry `queryImageUrl` and `matchRank`.

**Comments and buyer Q&A** (`goofish-xianyu-item-comments-scraper`):

```json
{ "startUrls": ["947872178004"], "maxCommentsPerItem": 50,
  "includeReplies": true, "includeQuestions": true }
```

- Billed per top-level comment; nested replies and the first Q&A page per item are free. `maxCommentsPerItem: 0` returns all.
- Comment rows nest seller answers in `replies[]`. Buyer Q&A goes to a separate dataset named `item-questions` (rows: `itemId`, `qid`, `question`, `answer`). If the run response does not list it under `storages.datasets`, find it with `get-dataset-list` and filter rows by `itemId`.
- Goofish has no per-listing star ratings. For transaction reviews, use the Seller tool.

## Reading results

Project fields with `get-dataset-items` `fields` (dot paths allowed). Good starting sets:

- Search and image search: `id,title,price,currency,city,province,condition.label,postedAt,stats.wants,seller.name,seller.stats.goodReviewRate,url`
- Item details: `rowStatus,id,title,price,description,conditionText,province,district,stats.views,stats.wants,seller.name,seller.stats.totalSold,seller.stats.goodReviewRate,url`
- Seller: `rowType,displayName,ipProvince,counts.items,counts.reviews,reviews,title,price,url`
- Comments: `itemId,authorRole,authorNick,content,postedAt,likes,replies`

`fetch-actor-details` returns the full field reference when the user asks for something outside these sets.

## Cost and limits

- Pricing is per result on each tool's Apify Store page (`https://apify.com/zen-studio/<name>/pricing`) and is billed to the user's Apify account. Quote the page rather than a remembered number.
- On Apify's free plan each tool allows a limited number of runs and caps rows per run (search: 30 listings). A capped run says so in its status message or a `_warning` row; tell the user the result is partial and that a paid Apify plan removes the cap.
- `maxItems`, `maxListings`, `maxReviews` and `maxCommentsPerItem` are the cost controls. State the cap you chose in the answer.
- Item details and comments return a status row (`rowStatus` invalid, unavailable, empty or error) for inputs that yield nothing; those rows are not billed.
