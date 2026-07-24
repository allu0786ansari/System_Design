# URL Shortener — Study Notes

A concept-by-concept guide to everything underlying this project. Written so
you can read a section, then go look at the actual code that uses it, and
have both make sense together.

**How to use this file:** don't try to memorize it. Read a section, then
find the matching part of the codebase, then come back and re-read the
section once you've seen it in action. The second read is where it
actually sinks in.

---

## Tier 0 — Foundations

Everything in this tier is not specific to this project. It's the general
computing knowledge the whole thing is built on top of. If an interviewer
asks "walk me through what happens when someone visits a short link," this
tier is what lets you actually answer instead of just describing your code.

---

### 0.1 Async / Await — Why This Whole Project Is `async def`

This is the single most important concept to understand before reading any
code in this project, because almost every function in the backend starts
with the word `async`.

#### The problem async solves

Imagine a single waiter working in a restaurant. A customer orders food.
The **slow** way to handle this: the waiter walks to the kitchen, stands
there, watches the food cook for 10 minutes doing absolutely nothing else,
then brings it out. While that's happening, no other customer can be
served — even customers who just want a glass of water, which takes two
seconds to serve.

The **fast** way: the waiter gives the order to the kitchen, then
immediately goes to serve other tables. When the kitchen finishes cooking,
it signals the waiter, who then delivers the food. One waiter, many tables,
nobody artificially waiting.

That's the entire idea behind async programming. A web server handling a
request often has to *wait* — for a database to respond, for Redis to
respond, for a network call to finish. Traditional ("synchronous") code
blocks the entire program during that wait. Async code lets the program
say "I'm waiting on this, but go do other useful work while you wait, and
come back to me when it's ready."

#### What `async` and `await` actually mean in Python

```python
async def get_link_by_slug(session, slug):
    result = await session.execute(select(Link).where(Link.slug == slug))
    return result.scalar_one_or_none()
```

- `async def` marks this function as one that *can* pause itself instead of
  blocking. It's called a **coroutine**.
- `await` marks the exact point where this function says "I'm about to do
  something slow (talk to the database) — pause me here, and let something
  else run in the meantime."
- Nothing here creates a new thread or a new process. It's all still one
  single thread. The "doing other things while waiting" happens through
  something called an **event loop** — a scheduler that keeps a list of
  paused coroutines and resumes each one the moment what it was waiting for
  becomes ready.

#### Why this matters for a URL shortener specifically

This project's whole justification for existing (see the caching layer,
Tier 2) is that redirects need to be *fast* and need to survive being hit
by a huge number of requests at once. If each request blocked the entire
server while waiting on Redis or Postgres, one slow database query would
freeze every other user's request too. With async, the server can have
hundreds of requests "in flight" — each one paused at its own `await`,
waiting on its own I/O — while a single CPU core keeps switching between
them efficiently.

**The one thing async does *not* do:** it doesn't make any single operation
faster. A database query that takes 5ms still takes 5ms. What async buys
you is the ability to handle *many* of those 5ms waits *concurrently*
instead of one at a time, back to back.

#### A concrete before/after

```python
# Synchronous (blocking) -- if this takes 50ms, the whole server is
# frozen for 50ms and cannot do anything else, for anyone.
def get_user(user_id):
    return db.execute("SELECT * FROM users WHERE id = ?", user_id)

# Asynchronous (non-blocking) -- while this 50ms database call is in
# flight, the event loop is free to start handling a different request.
async def get_user(user_id):
    return await db.execute("SELECT * FROM users WHERE id = ?", user_id)
```

**Interview-ready summary:** *"Async lets a single-threaded server handle
many requests concurrently by pausing a request at the exact point it's
waiting on I/O — a database call, a Redis call, a network call — and using
that idle time to make progress on other requests, instead of blocking the
whole process on one slow operation."*

---

### 0.2 HTTP Fundamentals

HTTP is the protocol (the agreed-upon set of rules) that a browser and a
server use to talk to each other. Every single interaction with this
project's API is an HTTP request followed by an HTTP response.

#### The request/response cycle

1. A client (a browser, `curl`, another program) sends a **request**: a
   method, a path, some headers, and optionally a body.
2. The server processes it.
3. The server sends back a **response**: a status code, some headers, and
   optionally a body.

That's it. Every API call in this entire project — creating a link,
redirecting, checking stats — is just this cycle, over and over.

#### HTTP methods used in this project

| Method | Meaning | Used for |
|---|---|---|
| `GET` | "Give me something, don't change anything." | Redirecting through a link, fetching stats, health checks |
| `POST` | "Create something new." | Creating a short link |

A method being **idempotent** means: calling it once has the same
end-effect as calling it 100 times. `GET` is idempotent — asking for the
same thing repeatedly doesn't change anything. `POST /links` in this
project is deliberately **not** idempotent — calling it twice with the
same URL creates two different links with two different slugs, on purpose
(so you can tell which channel a link was shared through, later, via
stats).

#### Status codes — and why each one used here was chosen deliberately

| Code | Meaning | Where it shows up |
|---|---|---|
| `200 OK` | Success, here's your data | Fetching link metadata, health checks |
| `201 Created` | Success, and I made a new thing | After `POST /links` |
| `307 Temporary Redirect` | "Go look somewhere else, but ask me again next time" | The actual short-link redirect |
| `400 Bad Request` | "Your request doesn't make sense" | A self-referential URL |
| `404 Not Found` | "That doesn't exist" | An unknown slug |
| `409 Conflict` | "That already exists, can't do this" | A custom alias that's already taken |
| `422 Unprocessable Entity` | "Your data is malformed" | A URL that isn't a valid URL |
| `429 Too Many Requests` | "Slow down" | Rate limiting |
| `500 Internal Server Error` | "Something broke on my end" | An unhandled bug |

Notice `307` specifically. A URL shortener could use `301 Moved
Permanently` instead — and many browsers would cache that redirect
forever, meaning the *browser itself* would stop even asking the server
after the first visit. That sounds efficient, but it breaks two things
this project cares about: it would make click analytics undercount (the
server never even sees the second visit), and it would make link
management impossible (if you deactivated a link, browsers that already
cached the old redirect would never find out). `307` avoids both problems
by telling the browser "this redirect might change, ask me again."

#### Headers

Headers are metadata attached to a request or response — not the "content"
itself, but information *about* it. A few relevant ones in this project:

- `Content-Type: application/json` — tells the server "the body of this
  request is JSON, parse it that way."
- `X-Request-ID` — a custom header this project adds to every response, so
  a specific request can be traced through logs later (see Tier 5).
- `Location` — sent back on a redirect response, telling the browser
  exactly where to go next.

---

### 0.3 REST API Basics

REST is a *style* of designing APIs, not a technology. The core idea: model
your system as a collection of **resources** (nouns), and use HTTP methods
(verbs) to act on them.

In this project, the resource is a **link**. So the API is designed as:

```
POST   /links              -> create a link
GET    /links/{slug}       -> fetch a link's metadata
GET    /links/{slug}/stats -> fetch a link's click stats
GET    /{slug}             -> actually redirect through a link
```

Notice the pattern: the *noun* (`links`) stays constant, and different
HTTP methods and sub-paths express different actions on it. This is the
opposite of an API that has endpoints like `/createLink` or `/getLinkInfo`
— those are technically fine, but they're not "RESTful," because the verb
is baked into the URL instead of expressed through the HTTP method.

#### Request and response bodies

Data sent to the server (in a `POST`) or sent back (in any response) is
usually JSON — a simple, human-readable format for structured data:

```json
// What the client sends to POST /links
{
  "target_url": "https://example.com/some/long/path",
  "custom_slug": "my-link"
}
```

```json
// What the server sends back
{
  "slug": "my-link",
  "short_url": "https://short.ly/my-link",
  "target_url": "https://example.com/some/long/path",
  "created_at": "2026-07-24T10:00:00Z",
  "is_active": true
}
```

The server doesn't just trust that incoming JSON is well-formed and safe —
that's what Pydantic validation (Tier 1) exists for.

---

### 0.4 Git Basics

You don't need to be a Git expert, but you need enough to understand *why*
this project's CI/CD pipeline exists at all (Tier 7 will go deeper on
CI/CD itself).

- **Repository (repo)**: the project's full history, tracked by Git.
- **Commit**: a saved snapshot of changes, with a message describing what
  changed and why.
- **Branch**: a parallel line of development — you make changes on a
  branch without touching the main, working version of the code
  (`main`) until you're confident they're ready.
- **Pull Request (PR)**: a request to merge your branch's changes into
  `main`, which is the natural point to run automated checks (tests,
  linting) *before* anything broken reaches the main codebase.

This last point is the entire reason CI exists: a PR is a checkpoint. This
project's CI workflow runs automatically the moment a PR is opened, and
merging is blocked until it passes — meaning nobody (including you) can
accidentally merge code that fails its own test suite.

---

## Tier 1 — Core API and Database (Phase 1 of the project)

Everything in this tier is what makes `POST /links` and `GET /{slug}`
actually work: turning an HTTP request into a validated Python object, into
a database row, and back into an HTTP response.

---

### 1.1 FastAPI — Routing and Dependency Injection

FastAPI is a Python web framework: it's the layer that turns "an HTTP
request arrived" into "call this specific Python function with these
specific arguments."

#### Routing

```python
@router.get("/{slug}")
async def redirect_to_target(slug: str, ...):
    ...
```

The `@router.get("/{slug}")` decorator tells FastAPI: "when a `GET`
request comes in matching this URL pattern, call this function, and
whatever value was in the `{slug}` part of the URL, pass it in as the
`slug` argument." This mapping from URL pattern to Python function is
called **routing**.

#### Dependency Injection — the more important concept here

This is the one FastAPI feature this project leans on constantly, and it's
worth understanding properly rather than just pattern-matching.

The problem it solves: almost every endpoint in this project needs a
database session, and some need a Redis connection, and some need
application settings. You *could* create these inside every single
function. But then:

- Every function has to know *how* to create a database session (tight
  coupling, repeated code).
- Tests become painful — to test one endpoint in isolation, you'd need a
  real database connection every time, because the connection logic is
  baked directly into the function.

FastAPI's `Depends()` solves this by *inverting* who's responsible for
creating things:

```python
async def redirect_to_target(
    slug: str,
    db: AsyncSession = Depends(get_db),
    cache: Redis = Depends(get_cache),
):
    ...
```

Here, `redirect_to_target` doesn't know or care *how* `db` and `cache` get
created. It just declares "I need a database session and a cache
connection," and FastAPI calls `get_db()` and `get_cache()` for you right
before running this function, then passes the results in.

**Why this matters for testing specifically:** because the function only
declares *what* it needs, not *how* to get it, tests can swap in something
different. This project's test suite overrides `get_db` to hand back a
fast, disposable in-memory SQLite session instead of a real Postgres
connection — and the endpoint code doesn't need to know or care that this
happened. That's the whole reason the test suite in this project can run
in under 2 seconds instead of needing a real database spun up for every
single test.

**Interview-ready summary:** *"Dependency injection means a function
declares what it needs as parameters, and something else — a framework, a
container — is responsible for constructing and providing those
dependencies. It decouples 'what do I need' from 'how do I get it,' which
is what makes the code both reusable and easy to test in isolation."*

---

### 1.2 Pydantic — Validating Data at the Boundary

Every request that reaches this API from the outside world is, from the
server's point of view, untrusted. A client could send malformed JSON, a
URL that isn't really a URL, a string where a number was expected, or
something malicious. Pydantic is the library that checks and enforces the
*shape* of data the moment it crosses that boundary — before any business
logic runs.

```python
class LinkCreateRequest(BaseModel):
    target_url: HttpUrl
    custom_slug: str | None = None
```

Just by declaring `target_url: HttpUrl` (instead of `target_url: str`),
Pydantic will:

- Reject the request with a `422` if `target_url` is missing.
- Reject it if the value isn't a syntactically valid URL at all.
- Automatically convert acceptable input into a proper `HttpUrl` object
  that the rest of the code can trust.

This project goes further with **custom validators** — for example,
rejecting a `custom_slug` that contains a space, or that's a reserved word
like `"health"` (which would otherwise silently break the `/health`
endpoint by shadowing it):

```python
@field_validator("custom_slug")
@classmethod
def _validate_custom_slug(cls, value):
    if not CUSTOM_SLUG_PATTERN.match(value):
        raise ValueError("custom_slug may only contain letters, numbers, hyphens, and underscores")
    return value
```

**Why validate here instead of later, deeper in the code?** Because the
earlier bad data is caught, the less of the system has to defend itself
against it. If validation only happened right before a database write, a
malformed URL could travel through cache logic, logging, and metrics
first — any of which might mishandle it. Catching it at the very first
line of defense means everything downstream can simply *trust* the shape
of the data it receives.

---

### 1.3 Relational Databases and SQL Basics

A relational database stores data in **tables** — think of a table as a
spreadsheet with strict rules about what can go in each column.

This project has two tables:

```
links                          clicks
+-- id (number, unique)        +-- id (number, unique)
+-- slug (text, unique)        +-- link_id (number, points to links.id)
+-- target_url (text)          +-- slug (text)
+-- is_active (true/false)     +-- referrer (text, optional)
+-- created_at (timestamp)     +-- user_agent (text, optional)
                                +-- created_at (timestamp)
```

#### Primary keys

Every table has a **primary key** — a column (usually `id`) guaranteed to
be unique and never null, which uniquely identifies one row. It's how you
say "this exact row, not any other."

#### Foreign keys

`clicks.link_id` is a **foreign key** pointing at `links.id`. This isn't
just documentation — the database itself *enforces* it: it will physically
refuse to insert a `Click` row whose `link_id` doesn't correspond to a
real, existing `Link` row. This is exactly the constraint that caught a
real bug in this project — an early version of the click-tracking code
tried to insert a click referencing a `link_id` that didn't actually
exist, and Postgres rejected it outright. (SQLite, used in the fast test
suite, doesn't enforce foreign keys by default — which is *why* that bug
was invisible until tested against real Postgres. This single fact is one
of the best interview stories in this whole project.)

#### Unique constraints and indexes

`links.slug` has a **unique constraint** — the database will refuse to let
two rows have the same slug, no exceptions, enforced at the storage layer
itself (not just "checked in Python code," which could have race
conditions — see Tier 4).

An **index** is a separate, fast-lookup structure the database maintains
alongside a table, so that looking up a row by a specific column (like
`slug`) doesn't require scanning every single row in the table one by one.
Without an index on `slug`, looking up a link by its slug would get slower
and slower as the table grew to millions of rows. With one, lookup time
stays roughly constant regardless of table size.

---

### 1.4 ORMs — Object-Relational Mapping

An ORM lets you interact with a database by writing Python objects instead
of raw SQL strings.

```python
# Without an ORM, you'd write raw SQL:
cursor.execute("SELECT * FROM links WHERE slug = %s", (slug,))

# With an ORM (SQLAlchemy, used in this project):
result = await session.execute(select(Link).where(Link.slug == slug))
```

`Link` here is a normal-looking Python class, but it's mapped onto the
`links` table — each instance of `Link` represents one row, and each
attribute (`Link.slug`, `Link.target_url`) represents one column.

**The tradeoff to understand, not just the convenience:** an ORM makes
common operations faster to write and less error-prone (no risk of typo'd
SQL, some protection against SQL injection by default). What it costs you
is a layer of abstraction between your code and the exact SQL that
actually runs — for very complex or performance-critical queries, you
sometimes need to know *what SQL the ORM is generating underneath* to
reason about performance properly. This project uses SQLAlchemy's
**async** mode specifically, meaning database calls integrate with the
`async`/`await` model from Tier 0 rather than blocking.

---

### 1.5 Database Migrations

A **migration** is a version-controlled, incremental change to a
database's schema (its structure — tables, columns, constraints) — the
database equivalent of a Git commit.

Why not just manually run `CREATE TABLE` statements on the production
database when you need a new table? Two reasons:

1. **No history.** If something goes wrong, there's no record of exactly
   what changed, when, or in what order.
2. **No repeatability.** You'd have to remember to make the exact same
   manual change on every environment — your laptop, staging, production
   — and any slip creates a mismatch between environments that causes bugs
   which are miserable to track down.

This project uses **Alembic** to manage migrations. Each migration file is
a small Python script with an `upgrade()` function (what to do) and a
`downgrade()` function (how to undo it):

```python
def upgrade() -> None:
    op.create_table(
        "links",
        sa.Column("id", sa.BigInteger(), primary_key=True),
        sa.Column("slug", sa.String(length=32), nullable=False),
        ...
    )

def downgrade() -> None:
    op.drop_table("links")
```

Migrations are applied in a strict, numbered order (this project's are
`0001_create_links_table.py`, then `0002_create_clicks_table.py`), and
every environment — your laptop, CI, production — runs the exact same
sequence, so the schema is guaranteed identical everywhere.

---

### 1.6 Database Sequences — the Atomicity That Makes Slugs Safe

A **sequence** is a database object whose entire job is to hand out
numbers, one at a time, and *guarantee* no two callers ever get the same
one — even if a thousand requests ask for a number at the exact same
instant.

```sql
SELECT nextval('link_id_seq');  -- always returns a fresh, never-repeated number
```

Why does this matter here specifically? Because this project needs to
allocate a unique numeric id for every new link, and it needs that
allocation to be safe **under real concurrency** — many people creating
links at the same moment. Handling this "safely" entirely in Python
(e.g., "check the highest existing id, add one") would have a **race
condition**: two requests could both check at the same moment, both see
the same highest id, and both try to use the same next id.

A database sequence avoids this because the *atomicity guarantee* — "this
operation cannot be interrupted or duplicated partway through" — is
enforced by the database engine itself, not by application code trying to
coordinate carefully. This is precisely what this project's real
end-to-end tests prove: firing 50 genuinely concurrent link-creation
requests and confirming zero duplicate ids ever come back.

---

### 1.7 Number Encoding — What Base62 Actually Is, and Why

You already know **base 10** — the normal number system, using 10 symbols
(`0`-`9`). Once you run out of single digits, you carry over into a second
digit (`9 -> 10`).

**Base 62** works identically, just with 62 symbols instead of 10:
`0`-`9`, then `a`-`z`, then `A`-`Z`. The *only* difference from base 10 is
how many symbols exist before you have to carry over into an extra digit
— which means numbers can be represented in far fewer characters.

Concretely: the number `238328` in base 10 is 6 characters. In base 62, it
encodes to `1000` — 4 characters — because each digit position in base 62
can represent 62 different values instead of just 10.

This is exactly why this project encodes a link's numeric database id into
base62 to produce its slug: it's not obfuscation or cleverness for its own
sake, it's purely a **compression technique** — representing the same
unique number in the fewest possible characters, using only symbols that
are safe to put directly into a URL without any special encoding
(`0-9a-zA-Z`, deliberately avoiding characters like `+`, `/`, or `=` that
would need escaping in a URL).

```python
# Simplified version of the actual logic
ALPHABET = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"

def encode(number):
    if number == 0:
        return ALPHABET[0]
    digits = []
    while number > 0:
        number, remainder = divmod(number, 62)
        digits.append(ALPHABET[remainder])
    return "".join(reversed(digits))
```

Because this is a true **bijection** (every distinct number maps to one
distinct string, and vice versa, with no overlaps), a sequence that never
repeats a number (Tier 1.6) guarantees the *slugs* generated from it never
repeat either — collision-safety by mathematical construction, not by
hoping a random string doesn't happen to clash with an existing one.

---

### 1.8 pytest Basics

Testing is how you prove code does what you claim it does — not just once,
by eye, but automatically, every time the code changes, forever.

#### The basic shape of a test

```python
def test_encode_decode_round_trip():
    slug = encode_slug(12345)
    assert decode_slug(slug) == 12345
```

A test is a function (pytest recognizes anything named `test_...`) that
does something and then **asserts** an expectation about the result. If
the assertion is false, the test fails, loudly, with the actual vs.
expected values shown.

#### Fixtures — reusable setup

Lots of tests need the same setup (a database session, a test client, some
sample data). A **fixture** is a reusable piece of setup code that pytest
injects into any test that asks for it, by name:

```python
@pytest.fixture
def db_session():
    # setup: create a fresh, empty in-memory database
    session = create_test_session()
    yield session
    # teardown: clean up after the test finishes
    session.close()

def test_create_link(db_session):
    # db_session is automatically provided here, already set up
    ...
```

This is the same underlying idea as FastAPI's dependency injection from
Tier 1.1 — declare what you need, let something else provide it — applied
to testing instead of to request handling.

#### Why this project's tests run against SQLite instead of real Postgres

Running a full Postgres server for every single test run would be slow and
would require infrastructure just to run `pytest`. So the main test suite
swaps in an in-memory SQLite database instead — fast, disposable, requires
nothing installed. The tradeoff (as Tier 1.3 already hinted at) is that
SQLite doesn't behave identically to Postgres in every respect — which is
exactly why this project also has a *separate*, smaller suite of tests
that run against a real Postgres instance, specifically to catch the
handful of things that only a real database would catch.

---

---

## Tier 2 — Caching (Phase 2 of the project)

Everything in this tier explains why `GET /{slug}` — the redirect — almost
never touches the database, and what makes that safe rather than just fast.

---

### 2.1 What Redis Actually Is

Redis is an **in-memory key-value store**. Two ideas packed into that
phrase, worth separating:

- **Key-value store**: the simplest possible database shape — you give it
  a key (a string, like `"slug:abc123"`) and a value, and later you ask
  for that same key back to get the value. No tables, no columns, no
  relationships between rows. Just lookups by key.
- **In-memory**: the data lives in RAM, not on disk.

That second point is the whole reason Redis exists. Reading from RAM is
dramatically faster than reading from disk — even a fast SSD is orders of
magnitude slower than RAM for random access. Postgres, by contrast, is
disk-backed (with some in-memory caching of its own, but its durability
guarantees mean it always ultimately writes to disk). Redis trades some of
that durability (data in RAM can vanish if the process crashes and isn't
configured to persist) for raw speed.

**Why that tradeoff is fine for this project specifically:** everything
stored in Redis here (a slug's target URL, a rate-limit counter) is either
disposable (if it's lost, just recompute it from Postgres, the real source
of truth) or short-lived by design (a rate-limit counter that resets every
minute anyway). Nothing irreplaceable is ever stored *only* in Redis.

---

### 2.2 The Cache-Aside Pattern

This is the single most important diagram to be able to draw on a
whiteboard in an interview.

```
                 ┌──────────────┐
   1. check ───► │    Redis     │
                 └──────┬───────┘
                         │
              2a. HIT ◄──┘──► 2b. MISS
                 │                │
          return immediately   3. fetch from Postgres
                                  │
                          4. write result into Redis
                                  │
                             return result
```

The name describes exactly what happens: the cache sits *beside* the real
data store, not in front of it as a mandatory gate. The application code
is responsible for checking the cache first, and only falling back to the
database — and then re-populating the cache — on a miss.

```python
async def redirect_to_target(slug, db, cache):
    cached = await get_cached_link(cache, slug)
    if cached.found:
        return redirect(cached.target_url)   # cache hit -- Postgres never touched

    link = await get_link_by_slug(db, slug)   # cache miss -- fall through
    await cache_positive(cache, slug, link.id, link.target_url, ttl_seconds=3600)
    return redirect(link.target_url)
```

**Why cache-aside specifically, instead of "always write to the cache the
moment data is created"** (a different, valid pattern called
write-through)? Because of the read/write ratio this whole project is
built around: a link might be created once and never clicked, or clicked a
million times. Pre-populating a cache entry for *every* created link,
whether or not it's ever visited, wastes cache memory on data nobody
actually reads. Cache-aside only ever caches data that's actually been
asked for at least once — with one deliberate exception in this project
(see 2.3).

**The one exception: write-through on creation.** This project's
`POST /links` endpoint *does* populate the cache immediately after
creating a link, rather than waiting for the first redirect to do it on a
miss:

```python
await cache_positive(cache, link.slug, link.id, link.target_url, ttl_seconds=settings.cache_ttl_seconds)
```

Why break the "only cache what's been asked for" rule here? Because a
freshly created link is very likely to be clicked within seconds — someone
just made it specifically to share it. Without this, that very first click
would pay the "cache miss, go hit Postgres" cost, defeating the entire
purpose of caching for the single click that's most likely to happen
soonest.

**Interview-ready summary:** *"Cache-aside means the application checks the
cache first, and only queries the source of truth on a miss, populating
the cache afterward. It's chosen over write-through when the workload is
read-heavy but unpredictable — you don't want to pay the cost of
pre-populating a cache entry for data that might never be read."*

---

### 2.3 TTL — Time To Live

A **TTL** is an expiration timer attached to a cached value. After it
elapses, Redis automatically deletes the entry — no application code has
to remember to clean it up.

```python
await cache.set(key, value, ex=3600)   # this entry auto-deletes after 1 hour
```

Why not just cache things forever? Two reasons:

1. **Staleness.** If the underlying data in Postgres ever changes (a link
   gets deactivated, for example), a cache with no expiration would keep
   serving the old, wrong answer indefinitely.
2. **Unbounded growth.** Without expiration, a cache holding data for
   every link ever created would grow forever, eventually exhausting
   available memory.

This project sets the TTL for a resolved link to **3600 seconds (1 hour)**
— long enough that a genuinely popular link stays fast without needing to
be re-fetched from Postgres every few minutes, short enough that if
something about the link ever needs to change in a future version, the
cache won't lie about it for more than an hour.

---

### 2.4 Negative Caching

This is the same idea as 2.3, applied to something you might not expect
you can cache: the *absence* of something.

When someone requests a slug that doesn't exist, the naive approach is:
check the cache (miss, since it was never cached — it doesn't exist),
check Postgres (also nothing, confirming it doesn't exist), return `404`.
If a bot or scraper hammers that same nonexistent slug 10,000 times a
second, *every single one* of those requests goes all the way to Postgres,
for a query that's guaranteed to fail every time.

Negative caching fixes this by caching the *fact* that a slug doesn't
exist, using a short-lived special marker:

```python
_NEGATIVE_SENTINEL = "\x00NOT_FOUND\x00"

async def cache_negative(cache, slug, *, ttl_seconds):
    await cache.set(_cache_key(slug), _NEGATIVE_SENTINEL, ex=ttl_seconds)
```

Now the second (and 10,000th) request for that same bad slug gets
rejected straight from Redis, never touching Postgres at all.

**Why the negative TTL (60 seconds) is so much shorter than the positive
TTL (3600 seconds):** a "not found" result is meant to absorb a *burst* of
repeated bad requests cheaply, not to be durable. If a slug that was
just confirmed missing gets created moments later, you don't want it stuck
returning a false `404` for a full hour just because it happened to be
checked once, milliseconds before it started existing.

---

### 2.5 Read-Heavy vs. Write-Heavy Workloads

This is less a specific technique and more the underlying *reason*
everything above exists at all — worth being able to state plainly.

A **write** changes data (creating a link). A **read** looks at data
without changing it (following a link, checking stats). Different systems
have wildly different ratios of one to the other, and that ratio should
drive your architecture.

A URL shortener is about as read-heavy as a system gets: a link is written
exactly once, and can be read an unbounded number of times afterward —
potentially millions, for a link that goes viral. Every caching decision
in this project traces back to that single fact: it's worth spending
effort making reads fast (caching, negative caching, indexes) precisely
*because* reads vastly outnumber writes, and it's fine for writes (link
creation) to be comparatively slower and more heavily guarded (rate
limiting, validation) precisely because there are so few of them by
comparison.

---

## Tier 3 — Async Processing (Phase 3 of the project)

Everything in this tier explains why clicking a link triggers analytics
tracking *without* the person clicking it ever having to wait for that
tracking to finish.

---

### 3.1 The General Problem: Don't Make Users Wait for Work They Don't Care About

When someone clicks a short link, two things need to happen: (1) redirect
them to the real URL, immediately, and (2) record that this click
happened, for later analytics. The person clicking the link cares
enormously about (1) being fast. They don't care *at all* whether (2)
finishes in 5 milliseconds or 5 seconds — they've already left the page.

If both of those happened directly inside the same request — redirect,
*then* write a row to a `clicks` table, *then* respond — the redirect
would be held up by however long the analytics write takes. Worse, if
recording the click ever failed (a database hiccup), it could break the
redirect entirely, even though the redirect itself had nothing wrong with
it.

The general fix, independent of any specific technology: split the work.
Do the thing the user is actually waiting on, then hand off the thing they
aren't waiting on to something else, entirely separately.

---

### 3.2 Message Queues and the Producer/Consumer Pattern

A **message queue** is exactly what it sounds like: a queue (first-in,
first-out, usually) that holds small messages until something picks them
up and processes them.

The pattern built around a queue has two independent sides:

- A **producer** puts messages onto the queue. It doesn't process them,
  doesn't wait for them to be processed, and doesn't even need to know
  what (or how many things) will eventually consume them.
- A **consumer** takes messages off the queue and does the actual work.

```
Redirect request ──► [produce a "click happened" message] ──► Queue
                                                                  │
                                          Worker process ◄───────┘ (consume, process)
```

**Why this decoupling matters, concretely:** the producer (the API,
handling the redirect) and the consumer (a separate worker process) can
now fail, restart, or scale completely independently. If the worker
crashes, redirects keep working perfectly — messages just pile up in the
queue until a worker comes back to process them. If click volume spikes,
you can run more worker processes without touching the API at all.

---

### 3.3 Celery: Tasks, Brokers, and Workers

**Celery** is a Python library that implements the producer/consumer
pattern described above, with specific vocabulary:

- A **task** is a unit of work — in this project, "save one click event to
  the database."
- A **broker** is the actual message queue infrastructure sitting between
  producer and consumer. This project uses Redis as the broker (the same
  Redis instance used for caching, but a separate logical database index,
  so cache keys and queue messages never mix).
- A **worker** is a separate, long-running process whose entire job is to
  pull tasks off the broker and execute them.

```python
# The API side (producer) -- just hands off a message, doesn't wait
def publish_click_event(link_id, slug, referrer, user_agent, occurred_at):
    celery_app.send_task("tasks.record_click", kwargs={...})

# The worker side (consumer) -- runs in a completely separate process
@celery_app.task(name="tasks.record_click")
def record_click(link_id, slug, referrer, user_agent, occurred_at):
    session.add(Click(link_id=link_id, slug=slug, ...))
    session.commit()
```

Notice these two pieces of code don't import each other, don't share a
codebase, and don't even run in the same process. The **only** thing
connecting them is that they agree, by convention, on the exact string
`"tasks.record_click"` and the exact shape of the arguments. This is
worth understanding precisely because it's a real, sharp edge: if that
string or those argument names ever drifted apart between the two
services, clicks would silently stop being recorded — no error would ever
surface, because the producer's job is just "hand the message to the
broker," which would still succeed even if no consumer understood it.

---

### 3.4 At-Least-Once Delivery and Idempotency

Most real message queues (Celery included) make a specific guarantee
called **at-least-once delivery**: a message will be delivered to a
consumer *at least* once — but might, in rare failure scenarios
(a worker crashing partway through processing, right before confirming
it finished), be delivered **twice**.

This is a deliberate design tradeoff. The alternative — guaranteeing
*exactly* once, no more, no less — is dramatically harder to build
correctly and usually isn't worth the cost for most workloads. Instead,
systems accept "maybe twice, rarely" and push the responsibility onto
whoever's *consuming* the message: make sure processing the same message
twice doesn't cause a problem. That property is called **idempotency**.

For this project's click-recording task specifically: inserting the same
click event twice would just mean one click gets counted twice in
analytics — a minor inaccuracy, not a broken system. This project accepts
that small risk rather than adding the extra complexity (like tracking
"have I already processed this exact message ID") that true exactly-once
processing would require. That's a legitimate, explicit tradeoff, not an
oversight — and being able to say *why* it's acceptable here (versus, say,
a payment-processing task, where double-processing would be a serious
bug) is a good signal in an interview.

---

### 3.5 Retries and Backoff

Sometimes a task fails for a reason that has nothing to do with the task
itself — a database connection hiccups for a second, a network blip. The
right response usually isn't "give up immediately" (that throws away work
for a problem that might already be over) or "retry instantly, forever"
(that can hammer an already-struggling system even harder).

This project's worker retries a failed click-recording task up to 3 times,
with a delay between attempts:

```python
@celery_app.task(name="tasks.record_click", bind=True, max_retries=3, default_retry_delay=5)
def record_click(self, **kwargs):
    try:
        ...
        session.commit()
    except Exception as exc:
        raise self.retry(exc=exc, max_retries=3)
```

`max_retries=3` caps how many times this will retry before giving up for
good — an important safety valve. Without a cap, a task that fails for a
*permanent* reason (not a transient blip) would retry forever, consuming
resources indefinitely for something that will never succeed.

The broker-level setting `task_acks_late=True` (configured on the worker,
not the task) adds one more layer of safety: a message is only removed
from the queue *after* the task finishes successfully, not the moment a
worker picks it up. If a worker process crashes mid-task, the unfinished
message goes back on the queue for a different worker to pick up — so a
crashed worker doesn't silently lose the work it was in the middle of.

---

---

## Tier 4 — Robustness (Phase 4 of the project)

Everything in this tier is about the gap between "this works when I test
it by hand" and "this survives real, messy, adversarial, concurrent use."

---

### 4.1 Race Conditions and Concurrency

A **race condition** happens when the correctness of a program depends on
the precise timing or ordering of events that are supposed to happen
independently — and that ordering isn't guaranteed.

Here's the classic, minimal example, using this project's own problem as
the illustration. Imagine (hypothetically — this project doesn't actually
do this, on purpose) slug generation was written like this:

```python
# DANGEROUS -- has a race condition
def generate_next_id():
    highest = db.query("SELECT MAX(id) FROM links")
    return highest + 1
```

If exactly one request runs this at a time, it works perfectly. But if two
requests run it at *nearly* the same instant:

```
Request A: reads highest = 999
Request B: reads highest = 999      <- both read the SAME value
Request A: computes 999 + 1 = 1000
Request B: computes 999 + 1 = 1000  <- both computed the SAME next id
```

Both requests now believe `1000` is their unique id. Whichever one writes
to the database second will either fail (if there's a uniqueness
constraint catching it) or, worse, silently overwrite or corrupt data (if
there isn't). The bug isn't in the *logic* — read the max, add one — the
bug is in the *assumption* that nothing else could possibly run that same
logic in the gap between the read and the write.

**Why this project avoids that entirely**, rather than trying to detect
and fix it after the fact: it uses a database sequence (Tier 1.6) instead,
where the "read the current value, then produce the next value" step is a
single, atomic, uninterruptible operation performed *by the database
engine itself* — there is no gap in the middle for a second request to
land in.

**Where this project still has to actively defend against races, rather
than sidestep them:** the bounded collision-retry logic in `POST /links`.
Even though a sequence-derived slug *should* be collision-free by
construction, the code doesn't simply assume that's true — it catches a
uniqueness violation if one somehow occurs and retries with a fresh id,
because "should never happen" and "provably cannot happen" are different
claims, and only the second one is safe to build zero error-handling
around.

**How this project actually *proves* it's safe, not just claims it:** the
end-to-end test suite fires 50 genuinely concurrent `POST /links` requests
— using `asyncio.gather` to launch them all at essentially the same
instant, over real, independent database connections — and asserts every
single one gets a distinct slug back. This is a meaningfully different
(and stronger) claim than a test that just calls the endpoint 50 times in
a row, one after another, which would never actually exercise the race
condition at all, even if one existed.

**Interview-ready summary:** *"A race condition occurs when correctness
depends on timing between concurrent operations that aren't properly
synchronized. The safest fix is to push the operation that needs to be
atomic down into something that guarantees atomicity — like a database
sequence — rather than trying to coordinate the timing carefully in
application code."*

---

### 4.2 Rate Limiting

Rate limiting means: cap how many requests a given client can make in a
given time window, and reject anything past that cap.

This project limits `POST /links` to **10 requests per minute per IP
address**, using a library (`slowapi`) backed by Redis:

```python
@limiter.limit("10/minute")
async def create_short_link(...):
    ...
```

**Why only the creation endpoint, and deliberately not the redirect
endpoint?** This is a good one to be able to explain unprompted, because
it reveals whether you actually understood the design or just copied a
pattern. Creation is the endpoint that *writes* to the database and
consumes a slot from a shared, finite sequence — exactly the kind of
operation you want to protect from being hammered (deliberately, by an
attacker, or accidentally, by a buggy client stuck in a retry loop).
Redirecting through an *existing* link, by contrast, is the entire reason
the system exists — a link that gets clicked heavily is the product
working correctly, not abuse. Rate-limiting redirects would actively
punish a link's success.

**Why the limit needs to live in Redis, not just in the API process's own
memory:** if this API ever runs as multiple replicas behind a load
balancer (a normal way to scale), an in-memory counter would mean each
replica enforces its *own* separate limit — a client could get 10 requests
per replica, effectively multiplying the real limit by however many
replicas exist. Storing the counter in Redis means every replica checks
and increments the *same* shared counter, so the limit is meaningful
regardless of how many API instances are running.

There are a few common algorithms for actually implementing a rate limit
— **fixed window** (count requests in, say, each clock-aligned minute, and
reset to zero at the top of the next minute) and **sliding window** or
**token bucket** variants (which avoid the fixed window's edge case where
a client could send double the limit by timing requests right around a
window boundary). Which exact algorithm `slowapi` uses under the hood
matters less for an interview than being able to explain *why* rate
limiting exists at all and *why this specific endpoint* was chosen.

---

### 4.3 Input Validation and Why You Never Trust Client Input

The general principle: any data arriving from outside your system —
typed by a user, sent by a script, forged by an attacker — must be treated
as untrusted until your own code has actively checked it. "It's just a
URL field" is exactly the kind of assumption that leads to real
vulnerabilities.

This project validates input at several distinct layers, each catching a
different category of problem:

**Format validation (Pydantic, Tier 1.2).** Is `target_url` even a
syntactically valid URL? Is `custom_slug` within the allowed length and
character set? This catches malformed *shape*, before any logic runs.

**Business-rule validation.** A `custom_slug` might be perfectly
well-formed as a string, but still unacceptable — for example, the value
`"health"` is a valid string by every formatting rule, but this project
explicitly rejects it because it would silently shadow the real
`/health` endpoint:

```python
RESERVED_SLUGS = frozenset({"links", "health", "metrics", "docs", ...})
if value.lower() in RESERVED_SLUGS:
    raise ValueError(f"'{value}' is a reserved word and can't be used as a custom_slug")
```

**Semantic / abuse validation.** A `target_url` might be a perfectly valid
URL, and not a reserved word, and still be a problem — specifically, if it
points back at the shortener's own domain, which could create an infinite
redirect loop or be used to disguise a malicious chain of redirects:

```python
if target_host.lower() == own_host.lower():
    raise HTTPException(400, "Cannot shorten a URL that points back at this shortener.")
```

**SQL injection, and why this project is protected from it almost by
accident of using an ORM.** SQL injection is an attack where malicious
input is crafted so that, if inserted directly into a raw SQL string, it
changes the *meaning* of the query instead of just being treated as data
— for example, input containing a stray quote character followed by
additional SQL commands. This project never builds SQL by concatenating
strings; the ORM (Tier 1.4) always sends parameters *separately* from the
query structure, so user input can never be interpreted as part of the
SQL syntax itself, no matter what characters it contains. This is worth
knowing not because this project had to think hard about it, but because
"how do you prevent SQL injection" is common enough that you should be
able to say precisely *why* the ORM protects you, not just that it does.

---

## Tier 5 — Observability (Phase 5 of the project)

Everything in this tier is about answering the question "how would you
know if this broke in production, and how would you find out why?" —
without observability, the only honest answer is "wait for someone to
complain."

---

### 5.1 Structured Logging

A traditional log line looks like this:

```
2026-07-24 10:00:00 INFO Created link abc123 for https://example.com
```

Perfectly readable by a human, skimming a file by eye. Almost useless to
a machine — extracting "which links were created in the last hour" out of
a wall of free-form text like that requires fragile text parsing.

**Structured logging** means every log entry is emitted as actual
structured data — this project uses JSON — instead of a free-form
sentence:

```json
{"timestamp": "2026-07-24T10:00:00Z", "level": "INFO", "logger": "app.routers.links", "message": "Created link", "request_id": "a1b2c3"}
```

Now the exact same information can be queried like a database: "show me
every `ERROR`-level log from the last hour," or "show me every log line
for this one specific `request_id`" — both of which are simple filters
over structured fields, not clever text-parsing. This matters far more at
real scale, once logs are being collected from many servers into a
centralized system, than it does reading a single log file on one laptop
— which is exactly why it's easy to underestimate until you've worked
somewhere that actually depends on it.

---

### 5.2 Metrics: Counters, Histograms, and the Cardinality Trap

A **metric** is a number tracked over time, meant to be aggregated and
graphed, not read line-by-line like a log.

Two basic shapes used in this project:

- A **counter** only ever goes up. `cache_lookups_total` is a counter — it
  counts how many cache lookups have happened, ever, and you derive a
  *rate* (lookups per second) by looking at how fast it's increasing over
  time, not by reading its raw value.
- A **histogram** buckets observed values into ranges, so you can ask
  questions like "what fraction of requests took under 50 milliseconds?"
  `http_request_duration_seconds` is a histogram tracking redirect
  latency.

#### Cardinality — the mistake that's worth understanding deeply

Every metric in this project is **labeled** — tagged with extra
dimensions you can filter and group by later. For example,
`http_requests_total` is labeled by `method`, `path`, and `status_code`.
Each *distinct combination* of label values creates a separate stored
timeseries.

Here's the trap: what if you labeled a metric by the raw URL slug instead
of the route pattern?

```python
# DON'T DO THIS
http_requests_total.labels(path=f"/{slug}").inc()   # slug is a raw, ever-growing value
```

Every single link ever created would generate a *brand new, permanent*
timeseries in Prometheus's storage — a system with a million links would
have a million distinct timeseries for this one metric alone, most of
which are only ever touched a handful of times. This is called a
**cardinality explosion**, and it's one of the most common, genuinely
production-breaking mistakes in observability — it can silently balloon
a monitoring system's memory usage until it falls over, often months after
the code that caused it was written and forgotten about.

This project avoids it by labeling with the route **template** instead of
the actual value:

```python
route = request.scope.get("route")
path_template = getattr(route, "path", None) or "unmatched"   # "/links/{slug}", not "/links/abc123"
```

Now there's exactly one timeseries per *type* of endpoint, regardless of
how many actual slugs exist — bounded, predictable, permanently small.

**Interview-ready summary:** *"Cardinality is the number of distinct label
combinations a metric can have. Labeling a metric by an unbounded value —
a user id, a raw URL, a slug — creates a new stored timeseries per value,
which can grow without limit and eventually exhaust the monitoring
system's memory. The fix is to label by a bounded category (like a route
template) instead of the raw value."*

---

### 5.3 Prometheus and Grafana

**Prometheus** works by **scraping**: on a fixed schedule (this project's
config uses every 15 seconds), it sends an HTTP request to a `/metrics`
endpoint exposed by the application, reads back a plain-text snapshot of
every current metric value, and stores that snapshot with a timestamp.
This is a *pull* model — the monitored application doesn't push data
anywhere; it just passively exposes its current numbers, and Prometheus
comes and collects them on its own schedule.

```
Prometheus  ──(every 15s, GET /metrics)──►  API
     │
     └── stores each scrape as a timestamped data point
```

**Grafana** is a separate tool that queries Prometheus's stored data and
renders it as dashboards. The query language is called **PromQL**. A
couple of real examples from this project's dashboard:

```
sum(rate(http_requests_total[1m])) by (path)
```
"Show me the request rate, in requests per second, over the last minute,
broken down by route." `rate()` is doing the work of turning an
ever-increasing counter into a meaningful per-second speed.

```
sum(rate(cache_lookups_total{result="hit"}[5m])) / sum(rate(cache_lookups_total[5m]))
```
"What fraction of cache lookups are hits?" — dividing one rate by another
to get a ratio, graphed continuously over time.

Being able to read a PromQL query and explain what question it's actually
answering (not necessarily write one from scratch) is a very reasonable
interview-level expectation.

---

### 5.4 Request Tracing via Request IDs

In a system handling many concurrent requests, log lines from many
different requests are naturally going to be interleaved in whatever
central place they end up. If something goes wrong for one specific user,
how do you find *only* the log lines that belong to their one request, out
of a firehose of everyone else's?

This project solves it by generating a unique **request ID** the moment
each request arrives, and making that ID available to every single log
line emitted anywhere during that request's handling — no matter how deep
in the call stack the log statement is:

```python
class RequestIDMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        request_id = request.headers.get("X-Request-ID") or str(uuid.uuid4())
        token = request_id_ctx_var.set(request_id)
        try:
            response = await call_next(request)
        finally:
            request_id_ctx_var.reset(token)
        response.headers["X-Request-ID"] = request_id
        return response
```

The mechanism making this possible — `contextvars` — is worth understanding
at a conceptual level: it's a way to attach a value to "the current
request's execution," retrievable from anywhere else running as part of
that same request, *without* having to explicitly pass that value as a
parameter through every single function call in between. It's specifically
designed to be safe under the concurrency model from Tier 0 — many
requests can be in flight on the same event loop simultaneously, each with
its own request ID, without them leaking into each other.

The `X-Request-ID` header is also **echoed back** in the response, and
reused (rather than replaced) if the client already sent one in. That
second detail matters in a real deployment: a load balancer or API gateway
sitting in front of this service might generate its own request id first
— reusing it here means one single ID can trace a request across every
system it passes through, not just this one.

---

---

## Tier 6 — Frontend (Phase 6 of the project)

Everything in this tier is what makes the React dashboard able to talk to
the API and update itself as data changes, without a full page reload.

---

### 6.1 React Basics — Components, Props, State, and Hooks

React's core idea: build a UI out of small, self-contained **components**,
each one responsible for rendering one piece of the page, that can be
combined together like building blocks.

#### Props — data passed *into* a component

```jsx
function LinkTag({ link }) {
  return <p>{link.short_url}</p>;
}
```

`link` here is a **prop** — data handed to this component by whatever
rendered it, the same way an argument is handed to a function. The
component has no say in what `link` contains; it just renders whatever
it's given.

#### State — data a component owns and can change over time

```jsx
const [url, setUrl] = useState("");
```

`useState` is a **hook** (a special React function, always starting with
`use`) that gives a component its own piece of memory that persists
between renders, plus a function (`setUrl`) to update it. Critically:
calling `setUrl(...)` doesn't just change a variable — it tells React "the
data this component depends on has changed, please re-render this
component with the new value." That re-render is what makes the UI
actually update on screen the moment someone types into the URL input.

#### `useEffect` — running code in response to something changing

```jsx
useEffect(() => {
  getLinkStats(slug).then(setStats);
}, [slug]);
```

This says: "every time `slug` changes, run this function." The array
`[slug]` at the end is the **dependency list** — it's what React checks to
decide whether to re-run the effect. This project's `LinkStats` component
uses exactly this pattern: expanding a different link in the registry log
changes the `slug` prop passed in, which triggers `useEffect` to fetch
that specific link's stats fresh, rather than showing stale data left over
from whichever link was expanded before.

**Why this specific pattern matters to understand, not just recognize:**
a very common React bug is forgetting an item in that dependency array —
the effect then keeps using an old, "stale" value from a previous render
instead of the current one. Understanding *why* the array exists at all
(so React knows when to re-run the effect) is what lets you actually debug
that class of bug instead of just copying a pattern that happens to work.

---

### 6.2 REST Client Patterns — How a Frontend Actually Talks to a Backend

The browser has a built-in `fetch()` function for making HTTP requests —
the client-side counterpart to everything described in Tier 0.2.

```javascript
const response = await fetch("/links", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ target_url: url }),
});
```

Notice `await` here too — `fetch()` is asynchronous for exactly the same
underlying reason server code is (Tier 0.1): a network request takes an
unpredictable amount of time, and the browser shouldn't freeze the entire
page while waiting for it.

**Why this project wraps every `fetch()` call in a single client module**
(`api/client.js`) instead of calling `fetch()` directly from inside
components: so that error handling, base-URL resolution, and response
parsing are handled consistently in exactly one place. Every possible
failure — a network error, a `429`, a `404` — gets translated into the
same shape of error (`ApiError`), so every component that calls the API
can handle failures the same simple way, without needing to know or care
whether the underlying problem was "the network dropped" or "the server
said no."

---

### 6.3 CORS — Cross-Origin Resource Sharing

This is a genuinely common interview question, and also a genuinely
common source of "why won't my frontend talk to my backend" confusion
during development — worth understanding precisely rather than just
knowing "add CORS headers" as a magic fix.

**The problem it solves:** browsers enforce a security rule called the
**same-origin policy** — by default, JavaScript running on one website
(one combination of scheme, domain, and port — an "origin") is *not*
allowed to make requests to a different origin and read the response.
This exists to stop a malicious website from silently making authenticated
requests to, say, your bank, using cookies your browser already has.

The dashboard (running on `http://localhost:5173` during development) and
the API (running on `http://localhost:8000`) are technically **different
origins** — different ports count as different origins even on the same
machine. Without anything special, the browser would block the
dashboard's JavaScript from reading the API's response at all.

**CORS is the mechanism that relaxes this, deliberately, on the server's
own terms.** The server includes specific response headers explicitly
listing which origins are allowed to receive its responses:

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.cors_allowed_origins_list,   # e.g. ["http://localhost:5173"]
)
```

This is an **allowlist**, not a blanket "allow everyone" switch — the
server is explicitly vouching for specific origins, and any origin not on
that list still gets blocked by the browser, exactly as before.

**Why the dev environment mostly avoids needing this at all:** Vite's dev
server proxy (6.4) makes the browser think it's only ever talking to one
origin (`localhost:5173`) by quietly forwarding `/api/*` requests to the
real API on the server side, where the browser's same-origin policy
doesn't apply at all — that policy is a *browser* rule, not a general
network rule.

---

### 6.4 Build Tools — What Vite Actually Does

Browsers can't run the project's source code directly as-is — modern
JavaScript/JSX needs to be **transformed** (JSX compiled into plain
JavaScript function calls) and **bundled** (many separate source files
combined into a small number of files the browser can efficiently load).
**Vite** is the tool that does both.

Two distinct things Vite provides, worth telling apart:

- **Dev server**: while you're actively developing, Vite serves your code
  with near-instant updates as you save files, without needing a full
  rebuild each time.
- **Production build** (`npm run build`): compiles everything down into a
  small set of optimized, minified static files — plain HTML, CSS, and
  JS — that can be served by any plain web server (this project uses
  nginx for that, in Tier 7).

The **dev-server proxy** mentioned in 6.3 is a Vite-specific convenience
configured in `vite.config.js`:

```javascript
server: {
  proxy: {
    "/api": { target: "http://localhost:8000", changeOrigin: true, rewrite: (path) => path.replace(/^\/api/, "") },
  },
}
```

This only exists during development. The production build has no such
proxy — it calls the real API's actual public address directly, which is
exactly why CORS has to be configured properly on the API for a real
deployment, even though it can be sidestepped entirely on a developer's
own laptop.

---

## Tier 7 — CI/CD and Deployment (Phase 7 of the project)

Everything in this tier is about the difference between "it works on my
machine" and "it's verified to work, automatically, every time, and can be
shipped safely."

---

### 7.1 Docker Fundamentals — Images vs. Containers

Two words that get used almost interchangeably in casual conversation but
mean genuinely different things:

- An **image** is a saved, read-only snapshot — everything needed to run
  a piece of software: the code, the exact language runtime version, all
  installed dependencies, filesystem layout. Think of it like a recipe, or
  a class definition.
- A **container** is a running *instance* of an image — think of it like
  an object instantiated from that class. You can start multiple
  containers from the exact same image.

**Why this matters more than it sounds like it should:** the entire point
is eliminating "works on my machine" bugs. Without containers, your app
depends on whatever Python version, whatever OS libraries, whatever exact
package versions happen to be installed on *your specific* laptop — which
might subtly differ from a teammate's laptop, or from the production
server. An image bundles the *entire* environment, not just the
application code, so the same image behaves identically regardless of
where a container from it actually runs.

This project's Dockerfiles use a **multi-stage build** — a "builder" stage
that installs dependencies and compiles things, and a separate, smaller
"runtime" stage that only copies over what's actually needed to *run* the
app:

```dockerfile
FROM python:3.12-slim AS builder
RUN pip install -r requirements.txt
...
FROM python:3.12-slim AS runtime
COPY --from=builder /opt/venv /opt/venv   # only the installed packages, not build tools
```

The benefit: the final image that actually gets deployed doesn't carry
around compilers, build caches, or other tools that were only ever needed
*during* the build — keeping the deployed image smaller and reducing its
attack surface.

---

### 7.2 Docker Compose — Running Multiple Containers Together

This project isn't one container — it's several (API, worker, dashboard,
Postgres, Redis, Prometheus, Grafana) that all need to run at once and
talk to each other. **Docker Compose** describes that entire multi-service
setup in one YAML file and starts/stops all of it together with a single
command.

```yaml
services:
  api:
    depends_on:
      postgres:
        condition: service_healthy   # wait until Postgres is actually ready, not just started
      redis:
        condition: service_healthy
```

Two ideas worth calling out specifically:

- **Service dependencies with health conditions**: `depends_on` alone only
  guarantees container *start order*, not that the dependency is actually
  ready to accept connections yet. `condition: service_healthy` waits for
  a defined **healthcheck** to actually pass first — otherwise the API
  container could start and immediately fail because Postgres, while
  technically "started," hadn't finished its own boot sequence yet.
- **A private network between services**: containers in the same Compose
  file can reach each other by service name (`postgres`, `redis`) as if
  those names were hostnames — that's why `DATABASE_URL` in this project
  points at `postgres`, not `localhost`, from inside a container.

---

### 7.3 CI/CD Concepts

**Continuous Integration (CI)**: automatically running your test suite
(and linting, and builds) every time code changes, rather than trusting
that a human remembered to test manually before merging.

**Continuous Deployment/Delivery (CD)**: automatically taking code that's
already passed CI and turning it into something deployable — building and
publishing container images, and (in a fuller setup) actually rolling them
out to a live environment.

**Why CI runs on every pull request, specifically** — not just
occasionally, or only before a release: the entire value of CI is
catching a problem at the *earliest, cheapest* possible moment. A broken
test caught on a PR costs one look at a CI failure. The same bug, caught
after it's merged and deployed, costs a rollback, a debugging session
under pressure, and possibly real user impact.

This project's CI is also **path-filtered** — a change confined to only
`services/dashboard/` doesn't spin up Postgres and Redis containers to run
the API's test suite, and vice versa:

```yaml
jobs:
  changes:
    outputs:
      api: ${{ steps.filter.outputs.api }}
  api:
    needs: changes
    if: needs.changes.outputs.api == 'true'
```

This is purely an efficiency optimization — running unrelated test suites
wouldn't be *wrong*, just wasteful of time and CI resources. It doesn't
change what's ultimately verified before a merge.

---

### 7.4 Environment Variables and Secrets Management

An **environment variable** is a piece of configuration passed to a
running program from *outside* its source code — a key-value pair the
operating system (or, here, Docker Compose) makes available to the
process.

```python
database_url: str = Field(default="postgresql+asyncpg://...")
```

**Why configuration lives outside the code at all**, rather than being
hardcoded: the exact same code needs to run against a local test database
on a laptop, a CI runner's temporary database, and a real production
database — none of which share a connection string. Environment variables
let *one* unchanged codebase behave correctly in every one of those
contexts, because the different values are supplied by whatever is
running it, not baked into the code itself.

**Why secrets (passwords, tokens) specifically must never be committed to
Git:** a Git repository's history is effectively permanent and often
widely accessible (especially once pushed to GitHub) — a password
committed once remains recoverable from the repository's history forever,
even if you "remove" it in a later commit, unless you rewrite history
entirely, which is disruptive and easy to get wrong. This project's
`.env.example` file documents every setting's *name* and *purpose*, with
placeholder or safely-default values, while the real `.env` file
containing actual credentials is explicitly excluded via `.gitignore` and
never committed.

---

### 7.5 The Testing Pyramid — and the Bug That Proves Why It Matters

The **testing pyramid** is a mental model for how a healthy test suite
should be shaped: many small, fast **unit tests** at the bottom, fewer
**integration tests** in the middle (testing multiple pieces together),
and a small number of **end-to-end tests** at the top (testing the real
system, as close to production conditions as practical).

The reasoning behind the shape: unit tests are fast and cheap, so you can
afford thousands of them, run on every single save. End-to-end tests are
slow and require real infrastructure, so you deliberately keep only a
handful — just enough to catch the specific class of bug that *only*
shows up under real conditions.

**This project hit exactly that class of bug, for real, which is worth
being able to narrate fluently:** the fast unit test suite runs against
SQLite instead of real Postgres, specifically for speed (Tier 1.8).
SQLite, by default, does not enforce foreign-key constraints. An earlier
version of the click-tracking logic could, for a custom link alias,
attempt to record a click referencing an id that didn't actually exist as
a real link — the SQLite-backed unit tests passed without any complaint,
because SQLite simply didn't check. Only the separate, smaller suite of
tests running against a *real* Postgres database caught it, because
Postgres does enforce that foreign key, and rejected the insert outright.

The fix wasn't "test more" in some vague sense — it was recognizing that
this specific category of bug (a database-engine-enforced constraint
behaving differently across database engines) can *only* be caught by a
test that uses the real engine, and structuring the test suite
deliberately to include exactly that, rather than assuming a fast test
suite alone was sufficient.

**Interview-ready summary:** *"A testing pyramid balances test speed
against realism — mostly fast unit tests for quick feedback, with a small
number of slower end-to-end tests specifically to catch bugs that only
appear under real infrastructure. In this project, a foreign-key
constraint violation was invisible in SQLite-backed unit tests, because
SQLite doesn't enforce foreign keys by default, and only surfaced once
tested against a real Postgres instance — which is precisely the class of
bug end-to-end tests exist to catch."*

---

## You've now covered every layer

Tier 0 through Tier 7 traces the exact same order this project was
actually built in: HTTP and async fundamentals, a working API and
database, a caching layer, decoupled async processing, hardening against
abuse and concurrency, observability, a frontend, and finally the
pipeline that tests and ships all of it.

The most useful next step isn't re-reading this file end to end — it's
picking any one bug or design decision mentioned here (the FK constraint
bug, the cache-decode issue with custom slugs, why 307 over 301, why
cache-aside over write-through) and practicing explaining it out loud, in
your own words, in under a minute, without looking at this document. That
is genuinely the skill an interview is testing — not whether you can
recognize these terms, but whether you can explain a real decision you
made and defend it.
