# Oxylabs docs errata

This file lists errors and gaps in the Oxylabs docs that live tests found, to report upstream.
GitBook exports the docs to `oxylabs/gitbook-public-english`, which is private, so the docs have no public issue tracker.
Each entry cites the page as published on 2026-09-25 and links the test behind it.

## Statements the API contradicts

### Response Codes: malformed JSON returns 400, not 422

- **Docs:** [Response Codes][response-codes] gives 422 for a bad payload: "Make sure it's a valid JSON object."
- **API:** malformed JSON returned `400 Bad Request` with `"message": "Json error: Unexpected value."`.
- **Evidence:** [Error bodies](docs/research/live-api.md#error-bodies).

### Response Codes: the content endpoint returns 204 for finished jobs

- **Docs:** [Response Codes][response-codes] describes 204 as "You are trying to retrieve a job that has not been completed yet."
- **API:** the content endpoint returned 204 for a `faulted` job, and for page numbers that a `done` job did not fetch.
  Only the undocumented `x-oxylabs-job-status` header tells these apart from a pending job.
- **Evidence:** [A faulted job](docs/research/live-api.md#a-faulted-job) and [The content endpoint](docs/research/live-api.md#the-content-endpoint).

### Response Codes: a 401 for an unknown username has no message

- **Docs:** [Response Codes][response-codes] lists three 401 messages: `Authorization header not provided`, `Invalid authorization header` and `Client not found`.
- **API:** a made-up username and password returned 401 with an empty body and `content-length: 0`.
  A request without the header returned the documented message.
- **Evidence:** [Error bodies](docs/research/live-api.md#error-bodies).

### Help center: 612 and 613 do not mean a failed submission

- **Docs:** [Response codes for Web Scraper API][help-response-codes] describes 612 and 613 as "Job submission failed."
- **API:** the submission returned 202, and 613 appeared later as the `status_code` of the faulted job's results entry.
  [Response Codes][response-codes] describes them correctly, as a job that Oxylabs failed.
- **Evidence:** [A faulted job](docs/research/live-api.md#a-faulted-job).

### Realtime: the output sample lacks the `job` object

- **Docs:** the [Realtime][realtime] output sample holds `results` only.
- **API:** every Realtime response held a `job` object beside `results`, with every Push-Pull job field except `_links`.
- **Evidence:** [A done Realtime job](docs/research/live-api.md#a-done-realtime-job).

### Push-Pull: `{n}` in the content endpoint is the page number itself

- **Docs:** [Push-Pull][push-pull] says "`{n}` is the page number starting at `1`."
- **API:** for a job with `start_page: 20` and `pages: 2`, `/results/20/content` and `/results/21/content` returned the pages, and `/results/1/content` returned 204.
  `{n}` matches the URLs in `href_list`, so it starts at 1 only when `start_page` is 1.
- **Evidence:** [The content endpoint](docs/research/live-api.md#the-content-endpoint).

### Push-Pull: the data dictionary types `statuses` and `client_id` wrongly

- **Docs:** the [Push-Pull][push-pull] data dictionary types `statuses` as Integer and `client_id` as String.
- **API:** `statuses` is a list, as the page's own samples show, and `client_id` is a JSON integer.
- **Evidence:** [Push-Pull submission](docs/research/live-api.md#push-pull-submission).

### Quick start: the 429 comes from a rate limit, not a concurrency limit

- **Docs:** [Quick Start][quick-start] describes 429 as "You have exceeded your concurrency limit."
- **API:** the limit counts submissions in a window of about one second.
  `-remaining` returned to 49 while earlier jobs were still `pending`.
  [Rate Limits][rate-limits] describes it correctly, as jobs per second.
- **Evidence:** [The window](docs/research/live-api.md#the-window).

### Usage Statistics: Realtime results count under `mode_callback_count`

- **Docs:** [Usage Statistics][usage-statistics] defines `mode_realtime_count` as "The amount of results, fulfilled via Realtime integration method."
- **API:** one Realtime result added 1 to `mode_callback_count` and left `mode_realtime_count` at 0.
  The fault may lie in the API rather than the docs.
- **Evidence:** [Deltas](docs/research/live-api.md#deltas).

### Usage Statistics: results are not split into HTML and parsed

- **Docs:** [Usage Statistics][usage-statistics] says source-level stats "are further broken down into separate statistics for HTML and parsed results."
  Its sample shows `contenttype_parsed_count` on each product and `parsed` on each source entry.
- **API:** neither field appeared, and two parsed `amazon_search` results counted under `contenttype_html_count`.
  The account predates 2024-09-25, so newer accounts may differ.
- **Evidence:** [Deltas](docs/research/live-api.md#deltas).

## Behaviour the docs leave out

### Response Codes: Realtime returns 408 past its TTL

- **Docs:** [Integration Methods][integration-methods] sets a 150-second TTL for every connection, and [Response Codes][response-codes] lists no 408.
- **API:** a Realtime job that ran past the TTL returned `408 Request Timeout` with `{"message":"Timed out."}` after 160 seconds, with no job ID.
- **Evidence:** [A Realtime job past the TTL](docs/research/live-api.md#a-realtime-job-past-the-ttl).

### Response Codes: submissions return 202, and Realtime returns 200 even for a faulted job

- **Docs:** [Response Codes][response-codes] lists both 200 and 202, and no page says which one a submission returns.
  [Realtime][realtime] does not say what a faulted job returns.
- **API:** Push-Pull and batch submissions returned `202 Accepted`.
  Realtime returned `200 OK` for a `done` job and for a `faulted` one, whose result carried `status_code: 613`.
- **Evidence:** [Status codes](docs/research/live-api.md#status-codes) and [A faulted Realtime job](docs/research/live-api.md#a-faulted-realtime-job).

### Response Codes: 613 appears only in the results entry

- **Docs:** [Response Codes][response-codes] lists 612 and 613 without saying where they appear.
- **API:** a faulted job kept `statuses` empty, and its results endpoint returned 200 with one entry whose `status_code` was 613.
  613 never appeared as an HTTP status, and no job showed 612.
- **Evidence:** [A faulted job](docs/research/live-api.md#a-faulted-job).

### Push-Pull: the results and content endpoints send `x-oxylabs-job-status`

- **Docs:** [Push-Pull][push-pull] does not mention the header.
- **API:** the results and content endpoints send `x-oxylabs-job-status` with `pending`, `done` or `faulted`.
  The content endpoint returns 204 for both a pending and a faulted job, so only this header tells them apart.
- **Evidence:** [A pending job](docs/research/live-api.md#a-pending-job) and [A faulted job](docs/research/live-api.md#a-faulted-job).

### Push-Pull: a batch lists invalid values under `errors`

- **Docs:** [Push-Pull][push-pull] shows only a batch whose values are all valid.
- **API:** a batch with one invalid `url` returned 202, created a job for the valid value, and listed the other under `errors` with its `message` and `url`.
  A batch whose every value failed also returned 202, with an empty `queries` list.
- **Evidence:** [A batch with invalid values](docs/research/live-api.md#a-batch-with-invalid-values).

### Rate Limits: each batch value and each page counts against the limit

- **Docs:** [Rate Limits][rate-limits] counts job submissions per second, and does not say how a batch or a job with `pages` above 1 counts.
- **API:** each batch value and each page of a `pages: 2` job took 1 from `-remaining`.
  A batch larger than `-remaining` returned 429 for the whole batch and created no job.
- **Evidence:** [What counts against the limit](docs/research/live-api.md#what-counts-against-the-limit) and [Exceeding the limit](docs/research/live-api.md#exceeding-the-limit).

### Rate Limits: the header names carry a UUID, and no header gives a reset time

- **Docs:** [Rate Limits][rate-limits] gives the pattern `x-ratelimit-limit_name-limit`.
  Its only example, a screenshot from December 2023, shows `x-ratelimit-internal-api-default-limit: 12000`.
- **API:** submissions returned `x-ratelimit-total-requests-<uuid>-limit: 50`, and rendered jobs added `x-ratelimit-total-render-requests-<uuid>-limit: 13`.
  Each came with a matching `-remaining` header.
  No response carried `Retry-After` or a reset header, including a 429.
- **Evidence:** [Limits and header names](docs/research/live-api.md#limits-and-header-names).

### Push-Pull: the content endpoint returns `png` as Base64 text

- **Docs:** [Push-Pull][push-pull] says the content endpoint returns a page "as raw content rather than inside a JSON object".
- **API:** for a `render: png` job, the content endpoint returned the Base64 string under `content-type: text/html`, not PNG bytes.
- **Evidence:** [The content endpoint](docs/research/live-api.md#the-content-endpoint).

[response-codes]: https://developers.oxylabs.io/products/web-scraper-api/response-codes
[help-response-codes]: https://developers.oxylabs.io/help-center/troubleshooting/response-codes-for-web-scraper-api
[integration-methods]: https://developers.oxylabs.io/products/web-scraper-api/integration-methods
[realtime]: https://developers.oxylabs.io/products/web-scraper-api/integration-methods/realtime
[push-pull]: https://developers.oxylabs.io/products/web-scraper-api/integration-methods/push-pull
[quick-start]: https://developers.oxylabs.io/get-started/quick-start-web-scraper-api
[rate-limits]: https://developers.oxylabs.io/products/web-scraper-api/usage-and-billing/rate-limits
[usage-statistics]: https://developers.oxylabs.io/products/web-scraper-api/usage-and-billing/usage-statistics
