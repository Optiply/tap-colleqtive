# tap-colleqtive

Singer tap for the [Colleqtive](https://colleqtive.com) public API, built with the [hotglue Singer SDK](https://hotglue.com).

Colleqtive is a mobile-first store operations platform that provides real-time store stock data. This tap extracts stock, product, order, and buy order data from the Colleqtive v2 public API.

---

## Streams

| Stream | Replication | Replication Key | Primary Key |
|---|---|---|---|
| `products` | Full Table | — | `product_number` |
| `stocks` | Incremental | `most_likely_datetime` | `id` |
| `orders` | Incremental | `last_modified_date` | `id` |
| `buy_orders` | Incremental | `last_modified_date` | `order_number`, `store_number` |

### Notes

- **products** — Full table sync on every run. The Colleqtive v2 API does not expose a `last_modified_date` field on the product endpoint, so incremental replication is not supported.
- **stocks** — Filters using the `last_stock_modified_datetime` query parameter on the API, but tracks state using the `most_likely_datetime` field from the response (the API does not return `last_stock_modified_datetime` in the response body).
- **orders** — Supports an optional `end_date` config key that maps to the `to_modified_date` API parameter with `reason_code=100`.
- **buy_orders** — Incremental by `last_modified_date`.

---

## Installation

```bash
pip install git+https://github.com/Optiply/optiply-integrations.git#subdirectory=taps/tap-colleqtive
```

Or with Poetry:

```bash
poetry add git+https://github.com/Optiply/optiply-integrations.git#subdirectory=taps/tap-colleqtive
```

---

## Configuration

Create a `config.json` file based on `config.json.example`:

```json
{
  "api_url": "https://app-api-bbq-cllqtv-prd-weu.azurewebsites.net",
  "token_url": "https://login.microsoftonline.com/<tenant-id>/oauth2/v2.0/token",
  "client_id": "your-client-id",
  "client_secret": "your-client-secret",
  "scope": "api://your-client-id.saascolleqtive.onmicrosoft.com/.default",
  "start_date": "2024-01-01T00:00:00Z",
  "page_size": 200,
  "requests_per_second": 4,
  "store_number": null
}
```

### Configuration reference

| Key | Required | Default | Description |
|---|---|---|---|
| `api_url` | Yes | — | Base URL of the Colleqtive API |
| `token_url` | Yes | — | OAuth2 token endpoint (Azure AD) |
| `client_id` | Yes | — | OAuth2 client ID |
| `client_secret` | Yes | — | OAuth2 client secret |
| `scope` | Yes | — | OAuth2 scope |
| `start_date` | No | — | ISO 8601 date — earliest date to sync for incremental streams |
| `page_size` | No | `200` | Number of records per API page (max 200) |
| `requests_per_second` | No | `4` | Rate limit for API requests |
| `request_timeout_seconds` | No | `120` | HTTP request timeout |
| `store_number` | No | `null` | Comma-separated list of store numbers to filter by. Leave null to sync all stores |

### Authentication

The Colleqtive API uses OAuth2 client credentials (Azure AD). The tap handles token acquisition and refresh automatically. Tokens are cached in memory and refreshed 60 seconds before expiry.

---

## Usage

### Discovery

```bash
tap-colleqtive --config config.json --discover > catalog.json
```

### Sync

```bash
tap-colleqtive --config config.json --catalog catalog.json | target-csv
```

### Incremental sync with state

```bash
tap-colleqtive --config config.json --catalog catalog.json --state state.json | target-csv >> state.json
```

---

## Pagination

The tap uses page-based pagination (`page_start` / `page_size`). It continues fetching pages until the number of records returned is less than `page_size`, then stops. The API response includes `total_count`, `next_page`, and `previous_page` fields for reference.

---

## Development

### Requirements

- Python `>=3.8,<3.12`
- [Poetry](https://python-poetry.org/)

### Setup

```bash
cd taps/tap-colleqtive
poetry install
```

### Running tests

```bash
poetry run pytest tests/
```

### Project structure

```text
tap-colleqtive/
├── tap_colleqtive/
│   ├── __init__.py
│   ├── client.py      # Authenticated HTTP client with OAuth2, rate limiting, and retry logic
│   ├── schemas.py     # Field definitions for all streams
│   ├── streams.py     # Stream definitions and pagination logic
│   └── tap.py         # Tap entry point
├── tests/
├── config.json.example
├── connector-config.json
└── pyproject.toml
```

---

## License

Apache 2.0 — see [LICENSE](../../LICENSE).
