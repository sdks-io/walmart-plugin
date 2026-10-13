---
name: "python-getting-started"
description: "Walmart APIs Python SDK identity and lookup layer (Python only) — install, import root, base URL/environments, the auth pattern, the SDK map that ships at the SDK root (`sdk-map.md` + `map/operations/`) and how to traverse it, and the module table naming the one file owning each fact the map leaves to the source. Load this before answering any Walmart APIs Python SDK contract question or writing any SDK code."
---

# Getting started with the Walmart APIs Python SDK

> **Who this skill is for.** This is the **lookup layer** for anyone writing Walmart APIs Python SDK code — it is yours to follow directly and fully. Ground every contract fact here (in the SDK map, and in the source modules it names) rather than in recall, and carry those facts onto a contract sheet before you implement. Load `python-integrate-walmart-apis` for the workflow that wraps this skill.

This is the **SDK-specific** entry point. For general patterns that apply to any APIMatic-generated Python SDK (client construction, auth, calling endpoints, models, error handling, resilience, testing), see the companion API-agnostic skills: `python-client-initialization`, `python-authentication`, `python-calling-endpoints`, `python-models`, `python-error-handling`, `python-configuration-resilience` and `python-testing`.

**This page and those companion skills are complementary — load both.** This page is authoritative for the SDK's *identity and surface* (what to install, what to import, the controllers, which module owns which fact); the companion skills are the *usage layer* on top — the best-practice way to call each piece and the gotchas a signature cannot show. Reading a module in the installed package does not remove the need to load the skill for that step, so at each step below, load the companion *and* confirm names against the installed package.

## SDK identity

Verified against `walmart_apis/` and `pyproject.toml` of the generated package at version `v4`. **Re-verify after a version bump** — this page is a snapshot, not a live read.

| Fact | Value |
| --- | --- |
| API | Walmart APIs |
| Distribution name (what you install) | `apimatic-wm-v4-sdk` |
| Import root (what you import) | `walmart_apis` — not the string you install |
| Source repository | https://github.com/sdks-io/apimatic-wm-v4-python-sdk |
| Source branch | `main` |
| Version | `v4` |
| Sync client class | `WalmartApisClient` (alias `Client`) |
| Async client class | `AsyncWalmartApisClient` (alias `AsyncClient`) |
| Client construction | **keyword-only**: `environment` · `timeout` (default `30.0`) · `server_config` · `retry_options` · `seller_auth_authorization_code` · `seller_auth_authorization_code_token_source` · `seller_auth_client_credentials` · `seller_auth_client_credentials_token_source` · `basic_client_auth` · `wallet_auth` · `wallet_auth_token_source`, and the transport override — `custom_http_client` on the sync client, `custom_async_http_client` on the async one (the names differ; see step 1) |
| Auth | **OAuth 2.0** authorization code — set `seller_auth_authorization_code` · **OAuth 2.0** client credentials — set `seller_auth_client_credentials` · HTTP **Basic** — set `basic_client_auth` · **OAuth 2.0** client credentials — set `wallet_auth` |
| Environments | 3 environments (default `"production"`) × 2 named servers, through `server_config` |
| Base-URL config | `ServerConfig` (`walmart_apis/server/server_config.py`), frozen, `extra="forbid"` |
| Python floor | **`>=3.10`** (classifiers list `3.10–3.14`) |
| Runtime dependencies | `httpx2 (>=2.13.1,<3.0.0)` · `certifi (>=2024.7.4)` · `pydantic[email] (>=2.11.0,<3.0.0)` · `typing-extensions (>=4.13.0,<5.0.0)` |
| Typing | ships `py.typed`; the package is checked under `mypy --strict` with `warn_unreachable`. Callers get full inference — **a type error against this SDK is a real contract violation, not noise** |
| Line length / lint | `ruff`, 120 cols (only relevant when editing the SDK itself) |
| Surface | 174 operations across 30 controllers · 410 models · 0 unions · 119 enums · 174 per-operation error unions |

The table above is **orientation, not a copy-paste recipe** — it gives you the names and facts (install, import roots, the auth *pattern*, the base-URL knob), while the actual integration code comes from the companion skills. Load each one as you reach its step (see **Integration workflow** below) and confirm its types against the installed package.

## Install

Into the same environment your project runs in, install the distribution from its package index:

```bash
pip install apimatic-wm-v4-sdk
python -c "import walmart_apis, pathlib; print(pathlib.Path(walmart_apis.__file__).parent)"
```

**Run the second line too.** A `pip install` that reports success is not proof the SDK is importable where your project runs. If the probe raises `ModuleNotFoundError`, the install went into a different interpreter: re-run it through the one your project uses (`python -m pip install …`) and probe again. If it prints a path whose contents are not this SDK, what you installed is not this distribution.

An index name can belong to an unrelated project — install it from its own repository instead, at <https://github.com/sdks-io/apimatic-wm-v4-python-sdk>:

```bash
pip install "apimatic-wm-v4-sdk @ git+https://github.com/sdks-io/apimatic-wm-v4-python-sdk.git@main"
```

The distribution carries its own `pyproject.toml`, so `pip` builds and installs it like any released package, copied into site-packages. Do not vendor its source into your project, add its directory to `sys.path`, or install it editable (`-e`) from a throwaway clone — an editable install points at that directory, so deleting the clone breaks every import.

Once the second line prints this SDK's directory, write the imports from the table above: the distribution name you install and the package name you import are not the same string. Requires Python 3.10 or newer.

## Imports — the package splits its surface across four modules

Python does not re-export child modules transitively, so `from walmart_apis import models` alone does **not** make enums, error unions, or runtime types reachable. Import each kind of type from the module that owns it.

`walmart_apis/__init__.py` exports exactly 8 names beside the `walmart_apis.models` subpackage it re-exports:

```python
from walmart_apis import (
    AsyncClient,
    AsyncWalmartApisClient,
    Client,
    Environment,
    ServerConfig,
    ServerConfigDict,
    ServerConfigOrDict,
    WalmartApisClient,
)
```

Everything else comes from its own subpackage, and the split matters because each kind of type a caller reaches for lives in a different module:

| What you need | Where it lives |
| --- | --- |
| Domain models, their `…Dict` companions | `walmart_apis.models` |
| Enums (and their open `…OrStr`/`…OrInt` aliases) | `walmart_apis.models.enums` |
| `ApiError` · `RawError` · `ApiResult` · `RequestOptions` · `AuthorizationCodeCredentials` · `ClientCredentials` · `BasicAuthCredentials` · `HttpClient` · `SdkBaseModel` · `UNSET` · `Optional` · `FileInput` · `AsyncFileInput` · `NamedFile` · `BinaryContent` · `AsyncBinaryContent` · `BinaryReader` · `AsyncBinaryReader` · `FileResponse` · `AsyncFileResponse` | `walmart_apis.core` |
| Per-operation error *unions* | `walmart_apis.errors` (`CreateWarrantyErrorBody`, …) |

`walmart_apis.core` re-exports its whole public surface (a curated `__all__`), so import from `…core` rather than from the private modules beneath it (`…core.results`, `…core.exceptions`, `…core.auth.schemes`).

## Environments and servers

The API declares 2 named servers across 3 environments, so the constructor takes `environment` first, then `timeout`, then `server_config: ServerConfigOrDict | None = None`. The environment selects which set of base URLs the config resolves against:

| `environment=` | Hosting |
| --- | --- |
| `"production"` *(default)* | Production |
| `"environment2"` | Sandbox |
| `"environment3"` | mock |

Each of the 2 servers resolves independently, so a call's host is the pair (server, environment):

| Server field | `"production"` base URL | `"environment2"` base URL | `"environment3"` base URL |
| --- | --- | --- | --- |
| `default` | `https://marketplace.walmart.com/seller` | `https://sandbox.marketplace.walmart.com/seller` | `https://walmart-apis-mock-server.onrender.com/seller` |
| `default1` | `https://marketplace.walmart.com/seller/v1` | `https://sandbox.marketplace.walmart.com/seller/v1` | `https://walmart-apis-mock-server.onrender.com/seller/v1` |

Consequences to state on every contract sheet that touches configuration:

- Omitting `environment` gives you **`"production"`**, silently.
- `server_config` overrides individual server URLs within the selected environment.
- The token endpoint (`/auth/v4/token`) resolves against the `default` server rather than a client-wide base URL, so `server_config` is what moves token traffic, and it follows `environment` with that server.
- The token endpoint (`/auth/token`) resolves against the `default1` server rather than a client-wide base URL, so `server_config` is what moves token traffic, and it follows `environment` with that server.
- `ServerConfig` is a frozen pydantic model with `extra="forbid"`: a misspelled keyword raises `ValidationError` at construction rather than being ignored.
- `timeout` is validated too — the base client raises `ValueError` for any non-positive value.

## Auth pattern (four schemes)

OAuth 2.0 authorization code, exposed as the client's `seller_auth_authorization_code=` keyword taking `AuthorizationCodeCredentials` **or a plain dict**. The client fetches and caches the bearer token itself, lazily, from `/auth/v4/token` on the `default` server.

```python
from walmart_apis import Client
from walmart_apis.core import AuthorizationCodeCredentials

def prompt(url: str) -> str:
    return input(f"Open {url}, then paste the code: ")

client = Client(seller_auth_authorization_code=AuthorizationCodeCredentials(client_id="…", client_secret="…", redirect_uri="…", prompt_for_authorization_code=prompt))
client = Client(seller_auth_authorization_code={"client_id": "…", "client_secret": "…", "redirect_uri": "…", "prompt_for_authorization_code": prompt})   # equivalent
```

OAuth 2.0 client credentials, exposed as the client's `seller_auth_client_credentials=` keyword taking `ClientCredentials` **or a plain dict**. The client fetches and caches the bearer token itself, lazily, from `/auth/v4/token` on the `default` server.

```python
from walmart_apis import Client
from walmart_apis.core import ClientCredentials

client = Client(seller_auth_client_credentials=ClientCredentials(client_id="…", client_secret="…"))
client = Client(seller_auth_client_credentials={"client_id": "…", "client_secret": "…"})   # equivalent
```

HTTP Basic authentication, exposed as the client's `basic_client_auth=` keyword taking `BasicAuthCredentials` **or a plain dict**.

```python
from walmart_apis import Client
from walmart_apis.core import BasicAuthCredentials

client = Client(basic_client_auth=BasicAuthCredentials(username="…", password="…"))
client = Client(basic_client_auth={"username": "…", "password": "…"})   # equivalent
```

OAuth 2.0 client credentials, exposed as the client's `wallet_auth=` keyword taking `ClientCredentials` **or a plain dict**. The client fetches and caches the bearer token itself, lazily, from `/auth/token` on the `default1` server.

```python
from walmart_apis import Client
from walmart_apis.core import ClientCredentials

client = Client(wallet_auth=ClientCredentials(client_id="…", client_secret="…"))
client = Client(wallet_auth={"client_id": "…", "client_secret": "…"})   # equivalent
```

**Every credentials keyword is optional at the type level and that is a trap worth flagging on every sheet.** Omit it and the client is built with `no_auth`: every request goes out unauthenticated. Nothing fails at construction. Most APIs then answer `401`; an API that serves anonymous traffic at all answers `200` and hides the omission entirely — verify the keyword is set rather than waiting for a `401` to tell you.

See `python-authentication` for the full picture, including what a *failed token fetch* raises — it is not what a caller expects, and it is the single most common surprise in this SDK.

## Controllers

Controllers and their operation counts (`client.<attr>`):

| Attribute | Class | Ops | Area |
| --- | --- | --- | --- |
| `client.authorization` | `Authorization` / `AsyncAuthorization` | 3 | `authorize` · `create_token` · `register_client` |
| `client.catalog` | `Catalog` / `AsyncCatalog` | 2 | `get_catalog_item` · `search_catalog_items` |
| `client.data_kiosk` | `DataKiosk` / `AsyncDataKiosk` | 5 | `cancel_query` · `create_query` · `get_document` · `get_queries` · `get_query` |
| `client.disputes` | `Disputes` / `AsyncDisputes` | 1 | `check_dispute_eligibility` |
| `client.feeds` | `Feeds` / `AsyncFeeds` | 5 | `cancel_feed` · `create_feed` · `get_feed` · `get_feed_document` · `get_feeds` |
| `client.final_payout` | `FinalPayout` / `AsyncFinalPayout` | 2 | `get_final_payout_case_status` · `update_final_payout_case_status` |
| `client.finances` | `Finances` / `AsyncFinances` | 4 | `list_financial_event_groups` · `list_financial_events` · `list_financial_events_by_group_id` · `list_financial_events_by_order_id` |
| `client.fulfillment_outbound` | `FulfillmentOutbound` / `AsyncFulfillmentOutbound` | 7 | `cancel_fulfillment_order` · `create_fulfillment_order` · `get_fulfillment_order` · `get_fulfillment_order_shipments` · `get_fulfillment_preview` · `list_all_fulfillment_orders` · … |
| `client.inventory` | `Inventory` / `AsyncInventory` | 4 | `bulk_update_inventory` · `get_inventory_for_sku` · `get_inventory_summaries` · `update_inventory_for_sku` |
| `client.listings_items` | `ListingsItems` / `AsyncListingsItems` | 5 | `delete_listings_item` · `get_listings_item` · `patch_listings_item` · `put_listings_item` · `search_listings_items` |
| `client.listings_restrictions` | `ListingsRestrictions` / `AsyncListingsRestrictions` | 1 | `get_listings_restrictions` |
| `client.merchant_fulfillment` | `MerchantFulfillment` / `AsyncMerchantFulfillment` | 5 | `cancel_shipment` · `create_shipment` · `get_eligible_shipping_services` · `get_label` · `get_shipment2` |
| `client.orders` | `Orders` / `AsyncOrders` | 14 | `acknowledge_order` · `cancel_order_lines` · `deliver_order_lines` · `get_order` · `get_order_address` · `get_order_buyer_info` · … |
| `client.payouts` | `Payouts` / `AsyncPayouts` | 5 | `configure_auto_payout` · `get_annual_statement_opts` · `get_faster_payout_eligibility` · `get_payment_status` · `initiate_faster_payout` |
| `client.product_type_definitions` | `ProductTypeDefinitions` / `AsyncProductTypeDefinitions` | 2 | `get_definitions_product_type` · `search_definitions_product_types` |
| `client.reconciliation` | `Reconciliation` / `AsyncReconciliation` | 3 | `check_download_report_by_period` · `download_reconciliation_report` · `list_reconciliation_dates` |
| `client.report_schedules` | `ReportSchedules` / `AsyncReportSchedules` | 4 | `cancel_report_schedule` · `create_report_schedule` · `get_report_schedule` · `get_report_schedules` |
| `client.reports` | `Reports` / `AsyncReports` | 7 | `cancel_report` · `create_report` · `download_new_report` · `download_old_report` · `get_report` · `get_report_document` · … |
| `client.sellers` | `Sellers` / `AsyncSellers` | 2 | `get_account` · `get_marketplace_participations` |
| `client.settlement` | `Settlement` / `AsyncSettlement` | 3 | `list_settlement_details` · `list_settlement_periods` · `search_transactions` |
| `client.uploads` | `Uploads` / `AsyncUploads` | 1 | `create_upload_destination_for_resource` |
| `client.wfs_inventory` | `WfsInventory` / `AsyncWfsInventory` | 1 | `get_wfs_inventory_items` |
| `client.messaging` | `Messaging` / `AsyncMessaging` | 11 | `create_warranty` · `get_attributes` · `confirm_customization_details` · `create_confirm_delivery_details` · `create_confirm_order_details` · `create_confirm_service_details` · … |
| `client.notifications` | `Notifications` / `AsyncNotifications` | 9 | `create_destination` · `create_subscription` · `delete_destination` · `delete_subscription_by_id` · `get_destination` · `get_destinations` · … |
| `client.product_fees` | `ProductFees` / `AsyncProductFees` | 3 | `get_my_fees_estimate_for_item` · `get_my_fees_estimate_for_sku` · `get_my_fees_estimates` |
| `client.product_pricing` | `ProductPricing` / `AsyncProductPricing` | 8 | `get_competitive_pricing` · `get_competitive_summary` · `get_featured_offer_expected_price_batch` · `get_item_offers` · `get_item_offers_batch` · `get_listing_offers` · … |
| `client.sales` | `Sales` / `AsyncSales` | 1 | `get_order_metrics` |
| `client.shipping` | `Shipping` / `AsyncShipping` | 9 | `cancel_shipment2` · `get_access_points` · `get_additional_inputs` · `get_rates` · `get_shipment_documents` · `get_tracking` · … |
| `client.solicitations` | `Solicitations` / `AsyncSolicitations` | 2 | `create_product_review_and_seller_feedback_solicitation` · `get_solicitation_actions_for_order` |
| `client.wfs_inbound` | `WfsInbound` / `AsyncWfsInbound` | 45 | `cancel_inbound_plan` · `cancel_self_ship_appointment` · `confirm_delivery_window_options` · `confirm_packing_option` · `confirm_placement_option` · `confirm_shipment_content_update_preview` · … |

Every controller has an `Async…` peer whose operations are identical in name and parameters and differ solely by being awaited. Do not emit a separate row for an async operation; state the rule once on the sheet.

The SDK map's controller table carries the same counts and links to a page per controller (`map/operations/<controller>.md`) — go there for the operations themselves.

## SDK map — look up first, open the module second

The SDK ships a generated map at its **root** — the directory holding `pyproject.toml`, the `walmart_apis/` source directory, and these two entries:

- **`sdk-map.md`** — the index: client construction with the full constructor-keyword table, the error-handling model (`ApiError` / `ApiResult` / `RawError`, Case A vs Case B), where models, enums and error aliases live, servers and auth, and the link table into the operations pages.
- **`map/operations/<controller>.md`** — one page per controller, one `###` block per operation: the HTTP verb and route, the sync parsed signature, each parameter's role and wire name, both return types, the error alias with the status each arm maps from, and a **Type sources** table naming the module that declares every *generated* type the operation mentions.

**Installing the distribution does not give you the map.** `pyproject.toml` ships the `walmart_apis/` package only, so `sdk-map.md` and `map/operations/` are absent from the installed package — they live in the SDK's own source tree, at the root of its repository, <https://github.com/sdks-io/apimatic-wm-v4-python-sdk>, in lockstep with the code the map describes — clone it to a temporary directory, outside the project repo:

```bash
git clone --depth 1 --branch main https://github.com/sdks-io/apimatic-wm-v4-python-sdk
```

Confirm the map comes from the release you installed before you read anything out of it: the clone above is pinned to `main`, the branch this SDK is released from — the repository's default branch may carry a different version. A map read from any other release describes code you do not have.

Every `Source` path on the map is relative to that SDK root, so `walmart_apis/models/access_point.py` opens as written from there — and the same path resolves inside the installed package, which is where you read a module's body once the map has named it.

**The map is the locator; the source modules are the shapes.** Read the map first — signatures, routes, parameter roles, return types, error unions, and which module declares a type are all answered there without opening a single `.py` file. Then open the one module the map names for what it deliberately does not carry: a model's members, an enum's values, a field's wire alias. The map says so itself — *"Shapes live only in the source … Never grep for a type."*

**The map carries shapes; what an operation *means* lives elsewhere.** When *what* to pass depends on meaning — which values a field accepts beyond its type, a rule that couples two fields, what a defaulted parameter actually selects — the map will not settle it. Open the operation's docstring in `walmart_apis/apis/<controller>.py` (or `client.py` where operations sit on the client) and `api-reference.md` at the SDK root for that operation *before* writing the sheet row, and record what you found there. A value you already "know" for a field the map types as a plain `str` is a lookup, not a recall — the memory ban applies to it.

`sdk-map.md` carries the invariants every operation block assumes, so load it before any `map/operations/` page; the pages are written to be read beside it.

## Contract facts — the map first, then the module

**Six of these are map lookups — don't open the module for them:** an operation's signature, parameters and return type; the client's constructor keywords; the `timeout` default; the members of `ApiError` / `Success` / `Failure` / `RawError`; an operation's error union and the status each arm maps from; base-URL and auth wiring. The table below covers everything else, and the full body behind a map row.

Read the one module that owns the fact **inside the installed package**. Locate it first:

```bash
python -c "import walmart_apis, pathlib; print(pathlib.Path(walmart_apis.__file__).parent)"
```

Failing that, it is under the project's environment (`.venv/Lib/site-packages/walmart_apis` on Windows, `.venv/lib/python3.*/site-packages/walmart_apis` elsewhere). **If the package is not installed, install it first** — see *Install* above; until then mark the fact `UNVERIFIED` and say what would settle it rather than answering from memory. Paths below are relative to that package root:

| Question | Module |
| --- | --- |
| An operation's real signature, parameters and return type | `apis/<controller>.py` |
| Client construction, keywords, controller wiring | `client.py`, `async_client.py`, `base_client.py` |
| Timeout default and validation | `base_client.py` (`DEFAULT_TIMEOUT = 30.0`) |
| The request/response pipeline, 401 handling, 2xx-vs-error split | `core/raw_client.py` |
| Exception shape (`ApiError.error`, `.status_code`, `.headers`) | `core/exceptions.py` |
| `Success`/`Failure`/`RawError` | `core/results.py` |
| Per-call overrides | `core/request_options.py` |
| `UNSET`, `Optional`, `OptionalNullable` | `core/optionality.py` |
| Model base config, `to_dict`/`to_json` | `core/models.py` |
| A model's members, required vs `UNSET`, wire aliases | `models/<model_name>.py` |
| An enum's members and wire values | `models/enums/` |
| Open-enum coercion | `core/converters/open_enum.py` |
| Date/time wire formats — `Date`, `RFC3339DateTime`, `RFC1123DateTime`, `UnixSecondsDateTime` (`Annotated` aliases over `datetime.date` / `datetime.datetime`; not in the map's Type sources) | `core/converters/date_time.py` |
| What a file or binary parameter accepts, and the reader protocols behind it (the aliases are in the import table above; not in the map's Type sources) | `core/files.py` |
| Consuming a downloaded body — `FileResponse` / `AsyncFileResponse`, and what closes it (not in the map's Type sources) | `core/file_responses.py` |
| Transport protocols (the test seam) | `core/transport.py` |
| httpx2 adapter, proxy/TLS knobs | `core/httpx2_transport.py` |
| Token fetch, credential placement | `core/auth/`, `core/auth/models.py` |
| Base-URL resolution | `server/server_config.py`, `server/server.py` |
| An operation's error mapper (status → schema) | `errors/<operation>_error.py` |

**Read scoped.** These modules carry long design docstrings; `grep -n` for the symbol and read the surrounding lines rather than whole files. Never quote a docstring's design rationale onto a contract sheet — the sheet carries facts an implementer must obey, not the reasoning behind them.

Keep lookups cheap — the rules that keep a session's context small:

- Collect the contracts for **every** in-scope operation in **one** pass — signature, required members with wire aliases, the error union, enum values — into a short **contract sheet** in your plan or working notes, then implement from the sheet. Don't re-open a module per member, and never re-look-up a fact the sheet already carries.
- Recurse into a model's members only where the task actually sets them — a full transitive expansion is hundreds of rows and nobody needs it.
- Trust the interpreter over this page: if a name here ever fails to type-check or import, re-read the module the table above names and report the drift; never patch around it from memory.

## Integration workflow — load the companion skill at each step

Before you write the code for each step, load the named companion skill — even if you have already read the relevant module. Each step calls out the trap the signature hides (in *parens*). A typical integration reaches them in this order:

1. **Client construction & lifetime** — load **python-client-initialization** before you write `Client(...)` or `AsyncClient(...)`. (*The signature won't tell you:* the constructor is keyword-only, so nothing can be passed positionally; the client owns an `httpx2` connection pool and you **must** `close()` (sync) or `await aclose()` (async) or use it as a context manager; it must be long-lived and module- or app-scoped, never rebuilt per request; the sync and async clients do not mix; and the transport-override keyword differs by client — `custom_http_client` vs `custom_async_http_client`.)
2. **Authentication** — load **python-authentication** before you set credentials. The four schemes are `seller_auth_authorization_code=`, `seller_auth_client_credentials=`, `basic_client_auth=` and `wallet_auth=`. (*The signature won't tell you:* every credentials keyword is *optional* — omit it and every request goes out unauthenticated, with no failure at construction and not necessarily a `401` to tell you; the token is fetched lazily and cached by the client; and a **failed token fetch** raises something other than what a caller expects and **bypasses the non-raising response mode** entirely. Load secrets from the environment or a secret store, never hardcode.)
3. **Calling an endpoint** — load **python-calling-endpoints** before the first `client.<controller>.<operation>(...)` call. (*The signature won't tell you:* every operation splits positional path params (and sometimes the body) from a keyword-only tail after `*`; every keyword-only parameter has a **real** default, so there is no "must pass `None` explicitly" hazard; **4 operations return `None`**, so `with_raw_response` is the only way to observe their status code; a multipart body is **one flat parameter list** — every field, JSON part and file is its own parameter, and an optional part is omitted with `None` rather than by a second call shape; and the two response modes — raising vs `ApiResult` — behave differently on failure.) (*For the file positions:* load **python-file-handling** before you pass anything but a `Path` — a file position accepts far more than `bytes`, **never a `str`**, and nothing validates what you hand it; an operation that returns a file hands back an **unread** `FileResponse` (`AsyncFileResponse` on the async client) that holds its connection until `save_to(...)`, `read()` or a drained `iter_bytes()` closes it, and a transport failure raises from that read rather than from the call.)
4. **Models** — load **python-models** the moment a request/response member is not a plain string or number. (*The signature won't tell you:* `Optional[T]` here is `T | UnsetType`, **not** `typing.Optional` — `None` is not a legal value for it; models are frozen pydantic instances with `…Dict` TypedDict companions; enums are **open** (`…OrStr`/`…OrInt`), so an unknown wire value passes through as a plain `str`/`int` rather than raising; wire aliases differ from Python member names; unknown response fields are **preserved**, not dropped; and serialize via `to_dict`/`to_json`.)
5. **Error handling** — load **python-error-handling** before you write any `try/except`. (*The signature won't tell you:* there is a single `ApiError` type whose `.error` is a **per-operation union**, and a decode failure raises `ValidationError`/`ValueError`, not `ApiError`, and bypasses both response modes; `httpx2` transport exceptions reach your boundary unwrapped; and for an operation that returns a file the call succeeds as soon as the head arrives, so a transport failure raises from the read rather than the call — put the boundary around the consuming code.)
6. **Configuration & resilience** — load **python-configuration-resilience** when you set the base URL, timeouts, retries, proxies, TLS, or logging. (*The signature won't tell you:* **retrying is on by default** — `retry_options` takes a count, a `RetryOptions` or its dict, `max_retries` set to `0` turns it off, and one call overrides it through `request_options`; a `401` is never retried; `timeout` defaults to `30.0` and is a single float that maps onto `httpx2`'s timeout semantics, limiting each wait rather than the whole call; and there is no logging hook — you wrap the transport seam.)
7. **Testing** — load **python-testing** before you stub the SDK. (*The signature won't tell you:* the seam is the **transport protocol** (`HttpClient`/`AsyncHttpClient` in `core/transport.py`) passed as `custom_http_client` — not the client class; a fake transport implements `send` and `close`, the two the protocol declares, and `send` answers with the head alone; assert on the request the SDK actually built, and cover all four failure kinds, decode failures included.)

## What a contract sheet must carry for this SDK

Beyond the usual signatures and model members, a Python sheet is incomplete without these, because each one is a decision the implementer cannot make correctly from the signature alone:

1. **Sync or async** — which client class, and the reminder that the two do not mix. Plus the `close()`/`aclose()` obligation and where the client is held.
2. **The keyword-only boundary** for each operation: what is positional (path params, sometimes the body) and what sits after `*`. Every keyword-only parameter has a real default, so there is no "must pass `None` explicitly" hazard — say so, so nobody writes defensive `None`s.
3. **The file positions** — `feeds.create_feed`, each annotated `FileInput` on the sync client and `AsyncFileInput` on the async one, with `list[…]` for an array part. What those admit is wider than the annotation shows and **never a `str`**, and the arm you pick decides what stays resident, who closes your handle and what the file is called on the wire — so the sheet cites **python-file-handling** for the row rather than restating it.
4. **The 4 operations that return `None`** — `authorization.authorize` · `shipping.submit_ndr_feedback` · `wfs_inbound.update_inbound_plan_name` · `wfs_inbound.update_shipment_name`. Their raw peers are `ApiResult[None, …]`, so `with_raw_response` is the only way to observe the status code.
5. **The 4 operations that return a file** — `feeds.get_feed_document` · `reconciliation.download_reconciliation_report` · `reports.download_new_report` · `reports.download_old_report`. The payload is `FileResponse` (`AsyncFileResponse` on the async client) with its body unread and its connection still held, so the sheet names the consuming call and where it is closed:
   1. `save_to(...)` · `read()` · a drained `iter_bytes()` — each consumes and closes. Getting it wrong leaks a connection rather than raising, so the row cites **python-file-handling** for the lifecycle: the async spellings, the single-use rule, and the warning an abandoned response emits.
   2. The raw peer observes the status and headers only — a `Success` carries no body of its own, because on a download the body *is* the payload.
6. **Required vs `UNSET`** for every model member the task sets, and the fact that `Optional[T]` here is `T | UnsetType` — **not** `typing.Optional`, so `None` is not a legal value for it.
7. **The `ApiError.error` union** for each operation in scope — there are **seven** typed error bodies in this SDK, so the union is never uniform. Every union is `<Typed> | RawError`:
   1. `ErrorList` — 104 operations (`catalog.get_catalog_item` · `catalog.search_catalog_items` · `data_kiosk.cancel_query` · `data_kiosk.create_query` · `data_kiosk.get_document` · `data_kiosk.get_queries` · `data_kiosk.get_query` · `disputes.check_dispute_eligibility` · `final_payout.get_final_payout_case_status` · `final_payout.update_final_payout_case_status` · `finances.list_financial_event_groups` · `finances.list_financial_events` · `finances.list_financial_events_by_group_id` · `finances.list_financial_events_by_order_id` · `fulfillment_outbound.cancel_fulfillment_order` · `fulfillment_outbound.create_fulfillment_order` · `fulfillment_outbound.get_fulfillment_order` · `fulfillment_outbound.get_fulfillment_order_shipments` · `fulfillment_outbound.get_fulfillment_preview` · `fulfillment_outbound.list_all_fulfillment_orders` · `fulfillment_outbound.update_fulfillment_order` · `inventory.bulk_update_inventory` · `inventory.get_inventory_for_sku` · `inventory.get_inventory_summaries` · `inventory.update_inventory_for_sku` · `listings_items.delete_listings_item` · `listings_items.get_listings_item` · `listings_items.patch_listings_item` · `listings_items.put_listings_item` · `listings_items.search_listings_items` · `listings_restrictions.get_listings_restrictions` · `merchant_fulfillment.cancel_shipment` · `merchant_fulfillment.create_shipment` · `merchant_fulfillment.get_eligible_shipping_services` · `merchant_fulfillment.get_label` · `merchant_fulfillment.get_shipment2` · `orders.acknowledge_order` · `orders.cancel_order_lines` · `orders.deliver_order_lines` · `orders.get_order` · `orders.get_order_address` · `orders.get_order_buyer_info` · `orders.get_order_items` · `orders.get_order_regulated_info` · `orders.get_orders` · `orders.refund_order_lines` · `orders.ship_order_lines` · `orders.ship_order_multi_package` · `orders.update_shipment_status` · `orders.update_verification_status` · `payouts.configure_auto_payout` · `payouts.get_annual_statement_opts` · `payouts.get_faster_payout_eligibility` · `payouts.get_payment_status` · `payouts.initiate_faster_payout` · `product_type_definitions.get_definitions_product_type` · `product_type_definitions.search_definitions_product_types` · `reconciliation.check_download_report_by_period` · `reconciliation.download_reconciliation_report` · `reconciliation.list_reconciliation_dates` · `report_schedules.cancel_report_schedule` · `report_schedules.create_report_schedule` · `report_schedules.get_report_schedule` · `report_schedules.get_report_schedules` · `reports.cancel_report` · `reports.create_report` · `reports.download_new_report` · `reports.download_old_report` · `reports.get_report` · `reports.get_report_document` · `reports.get_reports` · `sellers.get_account` · `sellers.get_marketplace_participations` · `settlement.list_settlement_details` · `settlement.list_settlement_periods` · `settlement.search_transactions` · `uploads.create_upload_destination_for_resource` · `wfs_inventory.get_wfs_inventory_items` · `messaging.create_warranty` · `messaging.get_attributes` · `messaging.confirm_customization_details` · `messaging.create_confirm_delivery_details` · `messaging.create_confirm_order_details` · `messaging.create_confirm_service_details` · `messaging.create_digital_access_key` · `messaging.create_legal_disclosure` · `messaging.create_unexpected_problem` · `messaging.get_messaging_actions_for_order` · `messaging.send_invoice` · `product_fees.get_my_fees_estimate_for_item` · `product_fees.get_my_fees_estimate_for_sku` · `product_fees.get_my_fees_estimates` · `sales.get_order_metrics` · `shipping.cancel_shipment2` · `shipping.get_access_points` · `shipping.get_additional_inputs` · `shipping.get_rates` · `shipping.get_shipment_documents` · `shipping.get_tracking` · `shipping.one_click_shipment` · `shipping.purchase_shipment` · `shipping.submit_ndr_feedback` · `solicitations.create_product_review_and_seller_feedback_solicitation` · `solicitations.get_solicitation_actions_for_order`); distinguishing members `errors`
   2. `CommonErrorList` — 45 operations (`wfs_inbound.cancel_inbound_plan` · `wfs_inbound.cancel_self_ship_appointment` · `wfs_inbound.confirm_delivery_window_options` · `wfs_inbound.confirm_packing_option` · `wfs_inbound.confirm_placement_option` · `wfs_inbound.confirm_shipment_content_update_preview` · `wfs_inbound.confirm_transportation_options` · `wfs_inbound.create_inbound_plan` · `wfs_inbound.create_marketplace_item_labels` · `wfs_inbound.generate_delivery_window_options` · `wfs_inbound.generate_packing_options` · `wfs_inbound.generate_placement_options` · `wfs_inbound.generate_self_ship_appointment_slots` · `wfs_inbound.generate_shipment_content_update_previews` · `wfs_inbound.generate_transportation_options` · `wfs_inbound.get_delivery_challan_document` · `wfs_inbound.get_inbound_operation_status` · `wfs_inbound.get_inbound_plan` · `wfs_inbound.get_self_ship_appointment_slots` · `wfs_inbound.get_shipment` · `wfs_inbound.get_shipment_content_update_preview` · `wfs_inbound.list_delivery_window_options` · `wfs_inbound.list_inbound_plan_boxes` · `wfs_inbound.list_inbound_plan_items` · `wfs_inbound.list_inbound_plan_pallets` · `wfs_inbound.list_inbound_plans` · `wfs_inbound.list_item_compliance_details` · `wfs_inbound.list_packing_group_boxes` · `wfs_inbound.list_packing_group_items` · `wfs_inbound.list_packing_options` · `wfs_inbound.list_placement_options` · `wfs_inbound.list_prep_details` · `wfs_inbound.list_shipment_boxes` · `wfs_inbound.list_shipment_content_update_previews` · `wfs_inbound.list_shipment_items` · `wfs_inbound.list_shipment_pallets` · `wfs_inbound.list_transportation_options` · `wfs_inbound.schedule_self_ship_appointment` · `wfs_inbound.set_packing_information` · `wfs_inbound.set_prep_details` · `wfs_inbound.update_inbound_plan_name` · `wfs_inbound.update_item_compliance_details` · `wfs_inbound.update_shipment_name` · `wfs_inbound.update_shipment_source_address` · `wfs_inbound.update_shipment_tracking_details`); distinguishing members `errors`
   3. `list[Error3]` — 8 operations (`product_pricing.get_competitive_pricing` · `product_pricing.get_competitive_summary` · `product_pricing.get_featured_offer_expected_price_batch` · `product_pricing.get_item_offers` · `product_pricing.get_item_offers_batch` · `product_pricing.get_listing_offers` · `product_pricing.get_listing_offers_batch` · `product_pricing.get_pricing`); distinguishing members *none required*
   4. `ErrorList1` — 5 operations (`feeds.cancel_feed` · `feeds.create_feed` · `feeds.get_feed` · `feeds.get_feed_document` · `feeds.get_feeds`); distinguishing members `errors`
   5. `OauthErrorModel` — 3 operations (`authorization.authorize` · `authorization.create_token` · `authorization.register_client`); distinguishing members `error`
   6. `AuthorizeError` — 1 operation (`authorization.authorize`); distinguishing members `error`
   7. `RegistrationError` — 1 operation (`authorization.register_client`); distinguishing members `error`
   8. *(none)* — `notifications.create_destination` · `notifications.create_subscription` · `notifications.delete_destination` · `notifications.delete_subscription_by_id` · `notifications.get_destination` · `notifications.get_destinations` · `notifications.get_subscription` · `notifications.get_subscription_by_id` · `notifications.send_test_notification`: no typed arm, so `.error` is always `RawError`
   9. So `isinstance(e.error, ErrorList)` matches only 104 of 174 operations.
   10. An arm that is not a class (`Any`, `list[...]`, `dict[...]`) is not an `isinstance` target: narrow the class arms first, then branch on `e.status_code` for what remains.
8. **That a decode failure raises `ValidationError`/`ValueError`, not `ApiError`, in both response modes** — `core/raw_client.py` states this in `execute`'s own docstring. **And that the 2xx path declares at least one required member on 64 return types, so a truncated body fails to decode there and passes silently everywhere else.** Any sheet row for a call whose result is used must name the members the implementer has to assert on.
9. **That retrying is on by default** — `retry_options` on the client tunes it, `max_retries` set to `0` turns it off, and `request_options` overrides it per call — so the sheet says whether this integration keeps, tunes or disables it, and for which methods, since a write is repeated only when the policy names its method.
10. **What limits a whole call.** `timeout` limits each wait, not the call, so a reply whose pieces each arrive within it is never cut off. In async code, state the budget and the one place `asyncio.wait_for` enforces it for every call, and what the boundary returns for its `asyncio.TimeoutError`; in sync code, say there is none.
11. **Which environment the `environment` keyword selects**, because omitting it is silently `"production"`.
12. A **REQUIRED READING** block naming the `python-*` companions that govern the steps, with `MUST load` pointers.

