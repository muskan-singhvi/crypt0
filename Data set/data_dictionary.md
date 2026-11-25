# Data Dictionary — CoinGecko Coin Market Data

> **Purpose:** Repository reference for the CoinGecko market-data API used as the crypto-asset data source. This document describes the API, its query parameters, the response structure, and every field observed in the supplied JSON sample.
>
> **Source identification:** The supplied payload is a JSON array of coin-market records and matches CoinGecko's **Coins List with Market Data** endpoint (`GET /coins/markets`). The original request URL and query string were not included with the payload, so the endpoint identification is based on the response schema. See the [official CoinGecko API reference](https://docs.coingecko.com/reference/coins-markets).

---

## 1. API overview

| Property | Description |
|---|---|
| Provider | CoinGecko |
| API product | CoinGecko API v3 — Coins |
| Endpoint | `GET /api/v3/coins/markets` |
| Public API URL | `https://api.coingecko.com/api/v3/coins/markets` |
| Pro API URL | `https://pro-api.coingecko.com/api/v3/coins/markets` |
| Response format | JSON (`application/json`) |
| Response root | Array of coin objects; one object per cryptocurrency |
| Main purpose | Retrieve current price, market capitalization, trading volume, supply, historical extremes, and price-change metrics for multiple coins |
| Primary record identifier | `id` (CoinGecko coin ID) |

The endpoint returns market data for multiple coins. The `vs_currency` query parameter determines the currency used for price, market-cap, volume, and other monetary values. The API documentation currently specifies USD as its default, but the supplied JSON does not contain the original request parameters; therefore, the quote currency used for this particular sample cannot be confirmed from the payload alone.

### Example request

This example requests the first 100 coins ordered by market capitalization and includes 24-hour and 7-day price-change percentages. Change `vs_currency` to the currency required by the application.

```bash
curl --request GET \
  --url 'https://api.coingecko.com/api/v3/coins/markets?vs_currency=usd&order=market_cap_desc&per_page=100&page=1&price_change_percentage=24h%2C7d'
```

For CoinGecko Pro, use the Pro base URL and provide the API key in the authentication header required by the account plan. Keep API keys in environment variables or a secret manager; do not commit credentials to the repository.

---

## 2. Request parameters

The following are the principal query parameters documented for this endpoint. Parameters that were used in the supplied request cannot be confirmed because its URL was not provided.

| Parameter | Type | Required | Description |
|---|---|---:|---|
| `vs_currency` | string | Yes | Quote currency for all monetary market data, such as `usd`, `inr`, or `eur`. Confirm and persist this value with each data load. |
| `ids` | string | No | Comma-separated CoinGecko coin IDs, for example `bitcoin,ethereum`. |
| `names` | string | No | Comma-separated coin names. URL-encode spaces. |
| `symbols` | string | No | Comma-separated coin symbols, for example `btc,eth`. Symbols may be shared by multiple assets. |
| `include_tokens` | string | No | For symbol lookups, `top` returns top-ranked matches and `all` returns all matching tokens. |
| `category` | string | No | Filters results to a CoinGecko coin category ID. |
| `order` | string | No | Sort order. Documented options include `market_cap_asc`, `market_cap_desc`, `volume_asc`, `volume_desc`, `id_asc`, and `id_desc`. Default: `market_cap_desc`. |
| `per_page` | integer | No | Number of records per page. Valid range: 1–250. Default: 100. |
| `page` | integer | No | Page number for pagination. Default: 1. |
| `sparkline` | boolean | No | Requests a 7-day sparkline object when `true`. Default: `false`. |
| `price_change_percentage` | string | No | Comma-separated percentage windows to include. Supported values: `1h`, `24h`, `7d`, `14d`, `30d`, `200d`, and `1y`. |
| `locale` | string | No | Language/localization setting for localized coin names where supported. Default: `en`. |
| `precision` | string | No | Decimal precision for currency price values. Supports `full` or a decimal-place value from `0` to `18`. |
| `include_rehypothecated` | boolean | No | When enabled, includes rehypothecated tokens in ranking and can add `market_cap_rank_with_rehypothecated` to the response. |

**Filter precedence:** When multiple filters are supplied, CoinGecko documents the precedence as `category` → `ids` → `names` → `symbols`. Use `page` and `per_page` for pagination. The API reference notes a maximum of 250 IDs per request. [See endpoint notes and parameter definitions](https://docs.coingecko.com/reference/coins-markets).

---

## 3. Response structure

The response is a top-level JSON array:

```json
[
  {
    "id": "bitcoin",
    "symbol": "btc",
    "name": "Bitcoin",
    "current_price": 58974,
    "market_cap": 1186425518319,
    "roi": null
  }
]
```

The example above is abbreviated. Each item represents one cryptocurrency. The supplied sample contains **100 records** and **28 top-level fields per record**. The following sections document those 28 fields.

### Type conventions

- `string`: JSON string.
- `number`: JSON numeric value. It may be an integer or decimal; use a decimal-capable type in a database for financial values.
- `integer`: whole-number value.
- `date-time`: ISO 8601 timestamp; timestamps in the sample use UTC (`Z`).
- `nullable`: the API may return JSON `null` when a value is unavailable or not applicable.
- Monetary values are denominated in the requested `vs_currency`; token supply values are denominated in units of the asset itself.
- Percentage values are expressed in percentage points. For example, `-2.5` represents a decrease of 2.5%, not `-0.025%`.

---

## 4. Column dictionary

### 4.1 Asset identity and presentation

| Field / column | API type | Description | Notes |
|---|---|---|---|
| `id` | string | CoinGecko's unique identifier for the cryptocurrency. | Recommended primary/business key for a coin. Example: `bitcoin`. |
| `symbol` | string | Trading ticker symbol for the cryptocurrency. | Example: `btc`. Do not use as a unique key; different assets can share a symbol. |
| `name` | string | Human-readable name of the cryptocurrency. | Example: `Bitcoin`. Names are labels, not identifiers. |
| `image` | string (URL) | URL of the coin's image or logo. | Treat as an external URL; it is presentation metadata, not a market measure. |

### 4.2 Price and trading activity

| Field / column | API type | Description | Notes |
|---|---|---|---|
| `current_price` | number \| null | Latest/current price of one unit of the coin in `vs_currency`. | Currency is determined by the request, not by the row itself. |
| `high_24h` | number \| null | Highest reported coin price over the previous 24 hours, in `vs_currency`. | Rolling 24-hour window. |
| `low_24h` | number \| null | Lowest reported coin price over the previous 24 hours, in `vs_currency`. | Rolling 24-hour window. |
| `total_volume` | number \| null | Total trading volume over the previous 24 hours, converted to `vs_currency`. | Monetary volume, not number of tokens traded. |
| `price_change_24h` | number \| null | Absolute price change over the previous 24 hours, in `vs_currency`. | Can be positive or negative. |
| `price_change_percentage_24h` | number \| null | Percentage change in price over the previous 24 hours. | Positive means price increased; negative means it decreased. |
| `price_change_percentage_24h_in_currency` | number \| null | 24-hour price-change percentage expressed against the selected quote currency. | Returned in the supplied sample; it is numerically the same as `price_change_percentage_24h` in the sample records. |
| `price_change_percentage_7d_in_currency` | number \| null | 7-day price-change percentage expressed against the selected quote currency. | Requires the relevant timeframe to be requested where applicable (for example, `price_change_percentage=7d`). |

### 4.3 Market capitalization and ranking

| Field / column | API type | Description | Notes |
|---|---|---|---|
| `market_cap` | number \| null | Market capitalization of the coin in `vs_currency`. | Monetary valuation reported by CoinGecko. |
| `market_cap_rank` | integer \| null | CoinGecko's market-cap ranking for the coin. | Lower rank number indicates a higher market-cap position. |
| `fully_diluted_valuation` | number \| null | Fully diluted valuation (FDV) of the coin in `vs_currency`. | A valuation measure based on the asset's fully diluted supply assumptions. |
| `market_cap_change_24h` | number \| null | Absolute change in market capitalization over the previous 24 hours, in `vs_currency`. | Can be positive or negative. |
| `market_cap_change_percentage_24h` | number \| null | Percentage change in market capitalization over the previous 24 hours. | Positive means market cap increased; negative means it decreased. |

### 4.4 Token supply

| Field / column | API type | Description | Notes |
|---|---|---|---|
| `circulating_supply` | number \| null | Estimated number of coins currently circulating in the market. | Measured in units of the token, not `vs_currency`. |
| `total_supply` | number \| null | Total supply of the token reported by CoinGecko. | Measured in units of the token. |
| `max_supply` | number \| null | Maximum supply limit for the token, when one is defined and available. | `null` may mean no known/defined maximum supply is reported. Do not treat `null` as zero. |

### 4.5 Historical price extremes

| Field / column | API type | Description | Notes |
|---|---|---|---|
| `ath` | number \| null | All-time high price of the coin in `vs_currency`. | ATH = all-time high. |
| `ath_change_percentage` | number \| null | Percentage difference between the current price and the all-time high. | Typically negative when the current price is below the ATH. |
| `ath_date` | date-time \| null | Date and time when the all-time high was recorded. | ISO 8601 timestamp; sample uses UTC. |
| `atl` | number \| null | All-time low price of the coin in `vs_currency`. | ATL = all-time low. |
| `atl_change_percentage` | number \| null | Percentage difference between the current price and the all-time low. | Typically positive when the current price is above the ATL. |
| `atl_date` | date-time \| null | Date and time when the all-time low was recorded. | ISO 8601 timestamp; sample uses UTC. |

### 4.6 Return on investment (nested object)

| Field / column | API type | Description | Notes |
|---|---|---|---|
| `roi` | object \| null | Return-on-investment information supplied by CoinGecko. | The entire object may be `null`. When present, it contains the child fields below. |
| `roi.times` | number | ROI multiplier. | Nested inside `roi`; for example, `43.3` is the multiplier value returned by the API. |
| `roi.currency` | string | Currency in which the ROI is expressed. | Nested inside `roi`; example values include `btc`, `eth`, or `usd`. |
| `roi.percentage` | number | ROI percentage. | Nested inside `roi`; percentage points, not a fraction. |

### 4.7 Data freshness

| Field / column | API type | Description | Notes |
|---|---|---|---|
| `last_updated` | date-time | Date and time when the coin's market data was last updated by CoinGecko. | ISO 8601 timestamp; sample uses UTC. This is the provider's update time, not necessarily the time your repository ingested the record. |

---

## 5. Nullability and observations from the supplied sample

The supplied JSON was inspected as a complete array:

| Observation | Result |
|---|---:|
| Number of coin records | 100 |
| Number of distinct top-level fields | 28 |
| Number of records with `max_supply: null` | 49 |
| Number of records with `roi: null` | 90 |
| `last_updated` value shared by all sample rows | `2026-11-25T07:26:54.000Z` |

The API schema permits `null` for several numeric and historical fields even when those fields happen to be populated in this particular sample. Database schemas and application code should therefore handle nullable values rather than assuming that a value will always be present. The counts above describe only the supplied snapshot; they are not permanent properties of the API.

---

## 6. Suggested repository / database conventions

1. **Use `id` as the asset key.** Do not use `symbol` or `name` as a primary key because they are not guaranteed to be unique.
2. **Persist the quote currency.** Store the `vs_currency` used for the request alongside the records or in the dataset metadata. A market price without its quote currency is ambiguous.
3. **Preserve precision.** Use a decimal/numeric database type for prices, market caps, volume, supplies, and percentage values. Avoid binary floating-point types for financial calculations where exact decimal representation matters.
4. **Keep source and ingestion timestamps distinct.** Preserve `last_updated` as the provider timestamp and add a separate `fetched_at` / `ingested_at` timestamp in the repository pipeline if historical snapshots are stored.
5. **Handle nullable values.** A `null` `max_supply` is not zero; a `null` `roi` means the nested ROI object is unavailable for that asset in the response.
6. **Flatten nested ROI only deliberately.** If a tabular warehouse is required, map `roi.times`, `roi.currency`, and `roi.percentage` to separate columns while retaining the ability to distinguish a missing `roi` object from a numeric zero.
7. **Record query metadata.** Store the request's `vs_currency`, filters, sort order, page, and retrieval timestamp so later analyses can reproduce how the dataset was collected.
8. **Respect pagination and response limits.** Use `per_page` and `page` when requesting more than one page; the documented maximum page size is 250.

### Suggested logical data types

| Data category | Suggested storage type |
|---|---|
| IDs, symbols, names, currency labels | `VARCHAR` / `TEXT` |
| Image URL | `TEXT` |
| Price, market cap, FDV, volume, supply, percentage and ROI numbers | `DECIMAL` / `NUMERIC` with precision appropriate to the asset range |
| Market-cap rank | `INTEGER` |
| `ath_date`, `atl_date`, `last_updated` | `TIMESTAMP WITH TIME ZONE` |
| `roi` | Separate nullable columns, or a JSON column if preserving the original structure is preferred |

---

## 7. Fields not present in this supplied payload

CoinGecko's endpoint can return additional fields depending on request parameters and API configuration. Examples documented by CoinGecko include:

- `market_cap_rank_with_rehypothecated`
- `sparkline_in_7d.price`
- `price_change_percentage_1h_in_currency`
- `price_change_percentage_14d_in_currency`
- `price_change_percentage_30d_in_currency`
- `price_change_percentage_200d_in_currency`
- `price_change_percentage_1y_in_currency`

These are **not included in the 28-field dictionary above** because they were not present in the supplied JSON sample. Add them to the repository schema only when the ingestion request enables or receives them.

---

## 8. References

- [CoinGecko — Coins List with Market Data (official API reference)](https://docs.coingecko.com/reference/coins-markets)
- [CoinGecko TypeScript SDK — generated `/coins/markets` resource and response types](https://github.com/coingecko/coingecko-typescript/blob/main/src/resources/coins/markets.ts)

**Document scope:** This dictionary describes the supplied 100-record response and the endpoint's documented behavior. It does not claim to know the original API key tier, exact request URL, quote currency, or all query parameters used to produce the sample.
