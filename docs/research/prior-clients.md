# Lessons from two earlier Oxylabs clients

This note compares two earlier Python clients for the Oxylabs Web Scraper API, as input for the rebuild.
Its keep and drop lists are standing preferences for the rebuild, unless a ticket on the map overturns one.
It describes design decisions and their consequences, not code, and the rebuild takes no code from either client.

Compared on 2026-09-23:

- **The earlier client** 1.0.0, an in-house package.
  It also held a data model, which this note leaves out.
- **`oxylabs` 3.0.0**, the official SDK ([oxylabs-sdk-python](https://github.com/oxylabs/oxylabs-sdk-python), released to PyPI on 2026-03-09).

The examples use a run of about 8,600 Amazon product jobs.
[What the docs state about the job lifecycle](job-lifecycle.md) holds the API facts, with sources.

## Summary

Neither client suits a Dagster pipeline as it stands.

The earlier client makes the right transport decisions: batch submission, pacing sized to the plan's rate limit, retries with backoff, and a status for every job.
It loses paid-for results when a run fails, though, and its main entry point reads and writes files instead of taking and returning data.

The SDK covers every Oxylabs target and parameter, and it passes unknown parameters straight to the API.
It has no batch submission, no pacing and no retries, and it returns an empty response for every kind of failure.
Its 3.0.0 release added targets and parameters; the code that sends requests, polls and handles errors is unchanged since 2.0.0.

A rebuild should keep the earlier client's transport decisions and the SDK's open parameter set.
It should add what both lack: a durable record of submitted jobs, bucket uploads with an upload check, and pacing that reads the API's rate-limit headers.

## Design decisions side by side

| Decision | Earlier client | Official SDK |
| --- | --- | --- |
| Integration methods | Push-Pull | Realtime, Push-Pull and Proxy |
| Sync or async | Async only | Separate sync and async clients |
| Submission | Batch endpoint, up to 50 jobs per request | One request per job |
| Pacing | Fixed, at most one batch per second | None |
| Retries | 429, 5xx and network errors, 10 attempts, exponential backoff | None |
| Polling | Submits every batch first, then polls up to 50 jobs at a time | Each call submits, polls and downloads one job |
| Default local timeout | 300 seconds per job | 50 seconds per job |
| Failure reporting | A terminal status per job; run-level errors raise | An empty response for any failure |
| Results | JSON files named by job ID, written after the last job finishes | Response objects in memory |
| Bucket uploads | Not possible | Possible through passthrough parameters |
| Inputs | A one-column CSV and a named YAML profile | Arguments to one method per target endpoint |
| Targets | Any source, by name | 46 targets |
| API parameters | Closed set; unknown keys raise | Named per method; unknown ones pass through |
| Credentials | Arguments, or environment variables loaded from `.env` | Arguments only |
| Logging | Module loggers; the CLI adds console and JSON-file output | Configures the root logger on import |
| Dependencies | httpx and tenacity, plus the data model's dependencies | aiohttp and requests |
| Type hints | Typed, ships `py.typed` | Partly typed, no `py.typed` |

## The earlier client: what to keep

- **Batch submission.**
  One request carries up to 50 jobs, so the 8,600-job run takes 172 submission requests instead of 8,600.
- **Pacing sized to the plan.**
  The batch size matches the account's jobs-per-second limit and shrinks for multi-page queries, so a single process stays under the limit.
- **Retries only for transient errors.**
  429s, server errors and network errors retry with exponential backoff, and client errors such as 400 or 401 fail at once.
  When two runs share the account's limit, the run that gets a 429 waits and retries instead of failing.
- **One retry policy for every call.**
  Submissions, status checks and result downloads all retry, so a transient error while polling does not abandon a running job.
- **A terminal status for every job.**
  Each job ends as done, faulted, timed out or errored, and the run returns all of them.
  One bad job does not fail the run, and the caller sees exactly which inputs failed.
- **Setup errors raise before anything is billed.**
  A missing input file, an invalid profile or missing credentials raise before the first submission.
- **Results named by job ID.**
  The ID is unique, so reruns never collide, and every file traces back to its Oxylabs job.
- **Atomic writes, with an optional local staging folder.**
  A reader never sees a half-written file.
  Staging on local disk first means an unreachable network drive leaves the files in staging instead of losing them.
- **Source-agnostic requests.**
  Any Oxylabs source works without a code change, because a request needs only the source name and its parameters.
- **A thin CLI.**
  The command line parses arguments and calls the same entry point that Python callers use, so the two behave the same.
- **A dry run.**
  The CLI lists what it would submit without calling the API, which is the cheapest check before a billed run.
- **Recorded API quirks.**
  The profile docs record which parameters fault on which targets, such as `geo_location` on `amazon_product` and `render: html` on Google search batches.
  Those notes came from billed test runs, so carry them into the rebuild.

## The earlier client: what to drop

- **Files as the interface.**
  The main entry point takes a CSV path, a YAML profile path and an output directory.
  A Dagster asset that holds its queries in a DataFrame has to write a CSV so the scraper can read it back.
  The lower-level client takes a list and returns results, which is the right shape for a pipeline, but it writes nothing and the docs present it as the secondary path.
- **Results held in memory until the run ends.**
  Nothing is written until every job reaches a terminal status, so a crash or Ctrl+C loses every result the run has paid for.
- **No record of accepted job IDs.**
  If any batch fails after its retries, or with a client error, the run raises before it polls anything.
  The jobs from earlier batches finish, are billed and stay retrievable for 24 hours, but no record of their IDs exists anywhere.
- **Polling waits for the last submission.**
  The 8,600-job run spends about three minutes submitting before it polls its first job.
- **Fixed pacing.**
  The client waits one second between batches, regardless of the rate-limit headers and of other processes using the account.
- **A retry schedule keyed to an undocumented header.**
  The client waits for the time in a `Retry-After` header when one is present, but the docs never mention that header, so plan around the exponential schedule.
  Ten attempts add up to about eight minutes of waiting before the client stops retrying.
- **One backoff for every 429.**
  An account-level 429 and a domain throttle get the same treatment, although the domain throttle affects one domain and lasts until its success rate recovers.
- **Pages assumed to count as jobs.**
  The client halves the batch for two-page queries, on the assumption that each page counts against the limit.
  The docs do not say whether it does.
- **A closed parameter set.**
  Profiles reject keys the client does not list.
  That catches typos, but every new API parameter needs a release; bucket uploads (`storage_type`, `storage_url`) are the current example.
- **Account limits set per profile.**
  Batch size, poll interval and poll concurrency sit in each scrape profile, next to the API parameters.
  The jobs-per-second limit belongs to the account, so every profile repeats it and any one of them can get it wrong.
- **Timed-out jobs are abandoned.**
  A job past the local timeout is marked timed out and never downloaded, but Oxylabs still finishes it and bills it.
- **An output fallback that hides failures.**
  When the output directory is unreachable, the CLI writes to a folder under Downloads and logs a warning.
  That keeps a manual run's data, but a pipeline step then succeeds while its files are somewhere else.
- **The library loads `.env`.**
  Loading environment files belongs to the application, and a library that does it changes the process environment as a side effect.
  In Dagster, credentials belong in a resource.
- **A naming rule for the input key.**
  The client derives the input key, `query` or `url`, from the source name.
  A new source whose name does not fit the rule gets the wrong key, and its requests fail.
- **One package for scraping and modelling.**
  Installing the scraper also installs polars, dataframely, dlt, marimo, rich and typer.

## Official SDK: what to keep

- **Coverage maintained by Oxylabs.**
  Each method lists its target's parameters with their documentation.
  That makes the SDK a useful reference for what the API accepts, even without using the client.
- **Unknown parameters pass through.**
  The target methods send extra keyword arguments straight to the API, so a new parameter works on the day Oxylabs adds it.
  This answers the earlier client's closed parameter set.
- **Explicit credentials.**
  The client takes a username and password and reads nothing from the environment.
- **One awaitable per job.**
  A caller can handle each result as soon as its job finishes, instead of waiting for the whole run.
- **A client identifier on every request.**
  Each request sends an SDK name and version header, which helps Oxylabs support trace a problem to its client.
- **Early checks on custom parsing instructions.**
  Malformed instructions raise before submission, so they never reach the API.
- **Realtime and Proxy methods.**
  For a few interactive calls, one synchronous request per job is simpler than submitting and polling.

## Official SDK: what to drop

- **No batch submission.**
  Every job is its own request, so the 8,600-job run takes 8,600 submission requests.
- **No pacing.**
  Concurrency is whatever the caller schedules, limited only by aiohttp's default pool of 100 connections.
  Scheduling a large list at once exceeds the rate limit in the first second.
- **No retries.**
  One 429, timeout or dropped connection ends the job.
  That includes status checks: a single failed poll abandons a job that is still running and will still be billed.
- **An empty response for every failure.**
  The client catches each error, logs it and returns an empty response.
  The caller cannot tell a 429 from a faulted job, a timeout or a network error, and nothing raises, so a run can lose most of its data and still finish normally.
- **Requests after a failed submission.**
  When a submission fails, the client still sends the status and result requests, for a job ID it never received.
- **A 50-second default timeout.**
  A job still running after 50 seconds is abandoned, and Oxylabs still finishes and bills it.
- **No persistence.**
  Results exist only in memory.
- **Response objects that mirror the JSON.**
  Attribute access covers only the fields the SDK maps, so anything else needs the raw dictionary, and every change to Oxylabs' output needs an SDK release.
- **The root logger configured on import.**
  Importing the package changes logging for the whole process, Dagster's included.
- **Two copies of every target.**
  The sync and async clients define each target separately, so every change has to be made twice.
- **Untyped for strict checking.**
  The package has partial type hints and no `py.typed` marker, so type checkers that follow PEP 561 treat it as untyped.
- **Two HTTP libraries.**
  The sync client uses requests and the async client uses aiohttp.
- **A major version that did not change the transport.**
  In the code that sends requests, polls and handles errors, 3.0.0 changed only type-hint syntax, so every point above also applies to 2.0.0.

## Lacking in both

- **A durable record of submitted jobs.**
  Neither writes job IDs down when the API accepts them.
  Results stay retrievable for 24 hours, so a record written at submission would let an interrupted run resume without submitting, and paying, again.
- **Coordination across processes.**
  Both pace within one process at best, while the limit applies to the whole account.
- **Pacing from the rate-limit headers.**
  Every submission response includes the remaining budget, and neither client reads it.
- **Handling for domain throttling.**
  Neither detects the domain throttle's 429, and neither stops submitting to a throttled domain.
- **Bucket uploads as a supported setting.**
  Neither treats uploads as a first-class option, and neither checks a job's upload status.
  A bucket permission error therefore goes unnoticed while every job's status stays `done`.
- **Recovery of timed-out jobs.**
  Both treat a local timeout as the end of a job, although Oxylabs finishes and bills it.
  A second pass over timed-out IDs within 24 hours would recover them.
- **A cost check before submitting.**
  Neither compares a run's job count with the plan's monthly result allowance.
- **A test seam.**
  Neither accepts a replacement HTTP layer, so testing code that calls them means patching the network or spending credits.
