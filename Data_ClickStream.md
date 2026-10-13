# Study Notes — Clickstream Lakehouse Project
### Topics 1–6: Programming & CS Foundations · Linux/Shell/CLI · Git & Version Control · Docker & Containerization · SQL · Data Engineering Core Concepts

Every concept below is explained from scratch, then immediately tied to
the exact file/line in the clickstream lakehouse project where it's
used — so you can flip between "what is this" and "where did I already
see this" without leaving this document. Later topics (7+) get appended
to this same file as they're covered, so this stays the single place to
study from.

---

# TOPIC 1 — Programming & CS Foundations

## 1.1 Python Fundamentals: Variables, Functions, Control Flow, Data Structures

### Variables & types
A variable is a name bound to a value. Python is dynamically typed — you
don't declare a type, but every value still has one.

```python
events_per_second = 5.0       # float
topic_name = "ecommerce.clickstream"   # str
is_late = False                # bool
```

**In this project:** `producer/config.py` is entirely variables holding
configuration:
```python
events_per_second: float = _float("EVENTS_PER_SECOND", 5.0)
```

### Functions
A function packages reusable logic. Parameters can have type hints and
default values.

```python
def add(a: int, b: int = 10) -> int:
    return a + b
```

**In this project:** almost every file is built from small, single-purpose
functions. Example from `spark_jobs/geo.py`:
```python
def ip_to_region(ip: str | None) -> str:
    if not ip:
        return "unknown"
    ...
```
Notice: one input, one output, no side effects — this is what makes it
trivially unit-testable in `tests/unit/test_geo.py` without needing Spark
or a database running.

### Control flow — `if`/`elif`/`else`, `for`, `while`
```python
for event_type in ("view_item", "add_to_cart", "purchase"):
    if event_type == "purchase":
        print("conversion!")
```

**In this project:** `spark_jobs/stream_ingest.py`'s validation logic is
control flow expressed as boolean column expressions instead of literal
`if` statements (because it runs across millions of Spark rows, not one
Python value at a time):
```python
is_valid = is_parseable & has_required_fields & has_valid_type
```
This is the same logical AND you'd write as `if a and b and c:` — just
vectorized across a DataFrame instead of one row.

### Data structures

| Structure | Mutable? | Ordered? | Example use in this project |
|---|---|---|---|
| `list` | yes | yes | `EVENT_TYPES = ("view_item", ...)` (actually a tuple, see below) |
| `tuple` | no | yes | `SessionPool._sessions` — list of `(user_id, session_id, ip)` tuples |
| `dict` | yes | yes (3.7+) | `EVENT_TYPE_WEIGHTS = {"view_item": 0.65, ...}` |
| `set` | yes | no | used in tests: `picks = {pool.pick() for _ in range(200)}` |

**Why a tuple, not a list, for `EVENT_TYPES`?** Tuples are immutable —
once defined, nothing downstream can accidentally append or remove an
event type. Using an immutable type for a "fixed set of valid values" is
a deliberate signal, not a stylistic accident.

```python
# producer/schemas.py
EVENT_TYPES = ("view_item", "add_to_cart", "purchase", "remove_from_cart")
```

### Dictionaries for configuration/weighting
```python
# producer/event_factory.py
EVENT_TYPE_WEIGHTS = {
    "view_item": 0.65,
    "add_to_cart": 0.20,
    "remove_from_cart": 0.05,
    "purchase": 0.10,
}
```
This dict is unpacked with `random.choices()`:
```python
types, weights = zip(*EVENT_TYPE_WEIGHTS.items())
return random.choices(types, weights=weights, k=1)[0]
```
`zip(*dict.items())` is a common Python idiom: `.items()` gives
`[("view_item", 0.65), ("add_to_cart", 0.20), ...]`, and `zip(*...)`
"transposes" that into two separate tuples: `("view_item", "add_to_cart", ...)`
and `(0.65, 0.20, ...)`.

---

## 1.2 Classes, Dataclasses, Type Hints

### Why classes at all?
A class bundles related data and behavior together. Instead of passing
around five loose variables (`event_id`, `event_type`, `user_id`...),
you pass around one object.

### `@dataclass` — Python's shortcut for data-holding classes
Writing a plain class to hold data means writing `__init__` by hand:
```python
class Point:
    def __init__(self, x: int, y: int):
        self.x = x
        self.y = y
```
`@dataclass` generates that boilerplate for you:
```python
from dataclasses import dataclass

@dataclass
class Point:
    x: int
    y: int
```

**In this project**, `producer/schemas.py`:
```python
@dataclass
class ClickstreamEvent:
    event_id: str
    event_type: str
    user_id: str
    session_id: str
    product_id: str
    category: str
    price: float
    event_timestamp: str
    ip_address: Optional[str] = None
    is_late: bool = False
```
Every field is a type-hinted attribute. `ip_address: Optional[str] = None`
means: this field is either a `str` or `None`, and defaults to `None` if
not provided.

### Type hints — what they actually do (and don't)
Type hints are **documentation Python checks at development time (with
tools like `mypy`) but does not enforce at runtime.** Writing
`price: float` doesn't stop you from passing a string — but it tells
every future reader (including future-you) exactly what's expected, and
lets your editor autocomplete and flag mistakes before you run anything.

```python
def ip_to_region(ip: str | None) -> str:
```
Read this as: "takes a string or `None`, always returns a string."
`str | None` is the modern (Python 3.10+) way to write what used to be
`Optional[str]`.

### `@dataclass(frozen=True)` — immutable config objects
`producer/config.py` and `spark_jobs/config.py` both use:
```python
@dataclass(frozen=True)
class ProducerConfig:
    bootstrap_servers: str = os.environ.get("KAFKA_BOOTSTRAP_SERVERS", "localhost:9092")
    ...

CONFIG = ProducerConfig()
```
`frozen=True` makes instances **immutable after creation** — trying
`CONFIG.topic = "something-else"` later in the code raises an error.
This is deliberate: config should be read once at startup and never
silently mutated somewhere deep in the code where a bug could quietly
change behavior.

### Static methods — `@staticmethod`
```python
@staticmethod
def new(event_type: str, ...) -> "ClickstreamEvent":
    ...
```
A static method belongs to the class but doesn't need `self` (an
instance) to run — it's called as `ClickstreamEvent.new(...)`, not
`some_event.new(...)`. It's used here as a **factory method**: a
named, controlled way to construct an object (generating the UUID,
validating `event_type`, defaulting the timestamp) instead of exposing
the raw dataclass constructor directly.

---

## 1.3 Modules, Packages, `__init__.py`, Imports

### Module vs package
- A **module** is a single `.py` file.
- A **package** is a folder containing an `__init__.py` file (even if
  empty) — that's what tells Python "treat this folder as importable."

**In this project:**
```
producer/
├── __init__.py       ← makes "producer" an importable package
├── schemas.py         ← module: import as `producer.schemas`
├── config.py
├── event_factory.py
├── kafka_producer.py
└── run_producer.py
```
Day 1 of Phase 2 explicitly created these:
```bash
touch producer/__init__.py tests/__init__.py tests/unit/__init__.py
```

### Absolute imports (used throughout this project)
```python
from producer.config import CONFIG
from producer.schemas import ClickstreamEvent
```
This says: "from the `producer` package, import the `CONFIG` object
defined in its `config` module." Absolute imports are preferred in this
project (over relative imports like `from .config import CONFIG`)
because they read unambiguously no matter which file you're looking at.

### The real-world import collision this project actually hit
Section on Airflow orchestration surfaced a genuine gotcha: this repo
has a top-level `airflow/` folder, but `apache-airflow` (the pip
package) is *also* imported as `airflow`. A plain
`import airflow.dags.clickstream_pipeline` would be ambiguous — Python
might resolve `airflow` to the local folder instead of the installed
library, breaking everything the DAG needs from the real Airflow
package.

**The fix used in this project** (`tests/unit/test_clickstream_pipeline_dag.py`):
load the DAG file directly by file path instead of by package import:
```python
import importlib.util

spec = importlib.util.spec_from_file_location("clickstream_pipeline_dag", DAG_PATH)
module = importlib.util.module_from_spec(spec)
spec.loader.exec_module(module)
```
This is worth understanding even outside this project: naming a local
folder the same as an installed library is a classic, hard-to-diagnose
bug.

---

## 1.4 Virtual Environments, pip, requirements.txt

### Why virtual environments exist
Different projects need different (sometimes conflicting) versions of
the same library. A virtual environment is an isolated Python
installation just for one project, so installing `pyspark==3.5.1` here
doesn't break some other project needing `pyspark==3.3.0`.

```bash
python -m venv .venv
source .venv/bin/activate    # on Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

### `requirements.txt` — pinned dependencies
**In this project**, `requirements.txt` evolved across phases:
```
kafka-python==2.0.2
faker==25.8.0
pyspark==3.5.1
delta-spark==3.2.0
psycopg2-binary==2.9.9
dbt-postgres==1.8.0
python-dotenv==1.0.1
pytest==8.3.2
apache-airflow==2.9.3
apache-airflow-providers-docker==3.13.0
docker==7.1.0
```
Every version is **pinned exactly** (`==`, not `>=`). This matters: an
unpinned `pyspark` could silently upgrade to a version with breaking API
changes (e.g., `dropDuplicatesWithinWatermark` didn't always exist) the
next time someone runs `pip install`. Pinning trades "always get the
latest" for "always get exactly what was tested."

### `pip install --break-system-packages` (containers/managed environments)
On some systems (and inside minimal Docker images), pip refuses to
install into the system Python without this flag, to avoid corrupting
OS-level Python tooling. You'll see this flag used when installing into
containers directly rather than a venv.

---

## 1.5 Context Managers (`with`)

A context manager guarantees cleanup code runs, even if an exception
happens in between — most commonly for closing files, connections, or
cursors.

```python
with open("file.txt") as f:
    data = f.read()
# file is automatically closed here, even if read() raised an exception
```

**In this project**, `spark_jobs/sync_silver_to_postgres.py`:
```python
conn = psycopg2.connect(...)
try:
    values = [tuple(getattr(r, c) for c in COLUMNS) for r in rows]
    with conn.cursor() as cur:
        execute_values(cur, INSERT_SQL, values, page_size=500)
    conn.commit()
finally:
    conn.close()
```
Two things happening here:
1. `with conn.cursor() as cur:` — the cursor is automatically closed
   after the `with` block, whether or not `execute_values` succeeds.
2. `try/finally` around the whole thing — guarantees `conn.close()`
   runs even if something inside raised an exception, preventing a
   leaked database connection.

## 1.6 Decorators (basic understanding)

A decorator wraps a function to add behavior without changing its code.
`@dataclass` (seen above) is itself a decorator. Another common one:

```python
from unittest.mock import patch

@patch("spark_jobs.sync_silver_to_postgres.psycopg2.connect")
def test_write_partition_skips_connection_for_empty_batch(mock_connect):
    _write_partition([])
    mock_connect.assert_not_called()
```
`@patch(...)` temporarily replaces `psycopg2.connect` with a fake
("mock") version for the duration of this one test function, then
restores the real one afterward — so the test doesn't need a real
Postgres connection to run.

## 1.7 Threading Basics

Threading lets multiple pieces of Python code run concurrently within
one process — useful when work is I/O-bound (waiting on network calls),
like sending events to Kafka.

**In this project**, `producer/run_producer.py`:
```python
threads = [
    threading.Thread(target=_worker, args=(per_thread_rate, session_pool, producer), daemon=True)
    for _ in range(args.threads)
]
for t in threads:
    t.start()
```
Each thread runs `_worker()` independently, generating and sending
events at a fraction of the total target rate. A shared `threading.Lock`
protects the counter every thread increments:
```python
_count_lock = threading.Lock()
with _count_lock:
    _sent_count += 1
```
**Why the lock matters:** without it, two threads could read
`_sent_count`, both compute `+1`, and both write back the same new
value — silently losing an increment. This is a **race condition**, and
the lock prevents it by ensuring only one thread touches `_sent_count`
at a time.

`daemon=True` means these threads are killed automatically when the
main program exits — you don't have to manually clean them up.

---

## 1.8 JSON — Structure & Serialization

JSON (JavaScript Object Notation) is a text format for structured data:
objects (`{}`), arrays (`[]`), strings, numbers, booleans, and `null`.

```json
{
  "event_id": "e1",
  "event_type": "view_item",
  "price": 19.99,
  "is_late": false
}
```

**Serialization** = converting a Python object into a string/bytes (to
send over the network, e.g., into Kafka). **Deserialization** = the
reverse.

**In this project**, `producer/schemas.py`:
```python
def to_json(self) -> bytes:
    return json.dumps(asdict(self)).encode("utf-8")
```
- `asdict(self)` converts the dataclass instance into a plain `dict`.
- `json.dumps(...)` serializes that dict into a JSON string.
- `.encode("utf-8")` converts the string into bytes, because Kafka
  transmits raw bytes, not Python strings.

On the Spark side, `spark_jobs/stream_ingest.py` does the reverse
(deserialization) using `from_json`:
```python
.withColumn("event", from_json(col("json_str"), schema))
```
This parses each JSON string back into a structured Spark column,
following the `CLICKSTREAM_EVENT_SCHEMA` defined in `spark_jobs/schema.py`.

---

## 1.9 Hashing, Partitioning, Windowing (Conceptual)

These three concepts show up constantly in data engineering and are
worth understanding independently of any one tool.

### Hashing
A hash function takes an input and deterministically produces a
fixed-size output (a number). The same input always produces the same
output. Kafka uses hashing internally: when you key a message by
`user_id`, Kafka hashes the key to decide which partition the message
goes to — which is *why* keying guarantees all of one user's messages
land on the same partition (same key → same hash → same partition,
every time).

### Partitioning
Splitting data into smaller, independent chunks so work can happen in
parallel. This project partitions in several places:
- **Kafka partitions** — the topic `ecommerce.clickstream` is created
  with 3 partitions (`docker exec ... kafka-topics --create ... --partitions 3`)
- **Delta table partitioning** — Silver is written with
  `.partitionBy("event_date")`, so queries filtering by date only scan
  relevant files instead of the entire table
- **Spark shuffle partitions** — `spark.sql.shuffle.partitions`
  controls how many chunks Spark splits data into during a `groupBy`

### Windowing
Grouping streaming data into time-based buckets so aggregation is
possible on data that never "ends." This project uses **tumbling
windows** (fixed, non-overlapping) — e.g., `window(col("event_timestamp"), "1 hour")`
in `spark_jobs/silver_to_gold.py` groups every event into exactly one
1-hour bucket, never two.

---

## 1.10 Big-O Intuition (Why Partition/Shuffle Sizing Matters)

You don't need to formally analyze this project's algorithms, but
understanding *why* certain choices were made requires basic intuition:

- A `groupBy` + aggregation over N rows, split across P partitions,
  costs roughly N/P work per partition **if data is evenly
  distributed**. If one partition gets way more data than others (data
  **skew**), that one partition becomes the bottleneck — the whole job
  waits on it. This is why Phase 3's proposal explicitly calls out
  "configure shuffle partitions to prevent data skewing."
- `dropDuplicatesWithinWatermark` deliberately keeps only a bounded
  window of state (not the whole table's history) specifically so
  memory usage doesn't grow linearly forever — an unbounded dedup would
  eventually run out of memory on a long-running job.

---

# TOPIC 2 — Linux, Shell & CLI

## 2.1 Navigating the Filesystem & Permissions

### Core navigation commands
```bash
pwd                  # print working directory
ls -la                # list files, including hidden ones, with details
cd spark_jobs         # change directory
mkdir -p a/b/c        # create nested directories in one go
```
`-p` ("parents") is what let Phase 1 create the entire project
skeleton in one command:
```bash
mkdir -p clickstream-lakehouse/{producer,spark_jobs,dbt,airflow/dags,data/bronze,data/silver,data/gold}
```
The `{a,b,c}` syntax is **brace expansion** — the shell expands this
into multiple `mkdir -p` targets before running the command.

### Permissions
Every file has read/write/execute permissions for owner/group/others.
```bash
chmod +x spark_jobs/submit.sh
```
This project uses `chmod +x` to make `submit.sh` **executable** — by
default, a newly created shell script isn't runnable directly
(`./submit.sh` would fail with "permission denied") until this is set.

**A permissions gotcha this project actually hit:** Airflow's container
couldn't read the mounted `/var/run/docker.sock` by default, because the
socket's ownership doesn't match Airflow's default container user. The
project's fix was `user: "0:0"` in `docker-compose.yml` — running the
container as root — explicitly documented as a local-dev trade-off, not
a production practice.

## 2.2 Shell Scripting Basics

```bash
#!/usr/bin/env bash
set -euo pipefail
```
This exact header appears in `spark_jobs/submit.sh` and `Makefile`
targets. Breaking it down:
- `#!/usr/bin/env bash` — the **shebang**, tells the OS which
  interpreter runs this file
- `set -e` — exit immediately if any command fails (instead of
  continuing and possibly doing more damage)
- `set -u` — treat using an undefined variable as an error
- `set -o pipefail` — if any command in a pipeline (`a | b`) fails, the
  whole pipeline is considered failed, not just the last command

**In this project**, `spark_jobs/submit.sh`:
```bash
#!/usr/bin/env bash
set -euo pipefail

docker exec -it lakehouse-spark-master spark-submit \
  --master spark://spark-master:7077 \
  --packages org.apache.spark:spark-sql-kafka-0-10_2.12:3.5.1 \
  /opt/spark-jobs/stream_ingest.py
```
The trailing `\` at the end of a line means "this command continues on
the next line" — purely for readability, it doesn't change behavior.

### Variables, conditionals in shell (used conceptually in Makefiles)
```bash
mode=$1                       # first argument passed to the script
if [ -z "$mode" ]; then
    mode="available_now"
fi
```
This pattern is why `spark_jobs/bronze_to_silver.py`'s Python
equivalent reads:
```python
mode = sys.argv[1] if len(sys.argv) > 1 else "available_now"
```
Same logic (default to a value if no argument was passed), just written
in Python instead of Bash.

## 2.3 Piping & Redirection

```bash
command1 | command2      # pipe: command1's output becomes command2's input
command > file.txt        # redirect: overwrite file.txt with output
command >> file.txt       # redirect: append to file.txt
```

**In this project**, injecting a malformed test message:
```bash
echo '{"event_id": "bad-1", "event_type": "not_a_real_type"}' | \
  docker exec -i lakehouse-kafka kafka-console-producer \
  --topic ecommerce.clickstream --bootstrap-server localhost:9092
```
`echo '...'` produces the JSON string; the `|` pipes that string as
**input** to `kafka-console-producer`, which reads from standard input
and publishes each line as a Kafka message. `docker exec -i` (note the
`-i`, "interactive") is required here — without it, the container's
standard input isn't connected, and the piped message would never reach
the process inside the container.

## 2.4 Environment Variables & `.env` Files

An environment variable is a named value available to a process and
everything it launches, without being hardcoded into source code.

```bash
export KAFKA_BOOTSTRAP_SERVERS=localhost:9092
```

**In this project**, `.env`:
```
POSTGRES_USER=lakehouse
POSTGRES_PASSWORD=lakehouse
POSTGRES_DB=warehouse
SPARK_WORKER_MEMORY=2G
SPARK_WORKER_CORES=2
```
Docker Compose automatically reads `.env` in the same directory and
substitutes `${POSTGRES_USER}` etc. inside `docker-compose.yml`. Python
reads environment variables via `os.environ.get(...)`:
```python
# spark_jobs/config.py
kafka_bootstrap_servers: str = os.environ.get("KAFKA_BOOTSTRAP_SERVERS", "kafka:29092")
```
The second argument (`"kafka:29092"`) is the **default** used if the
environment variable isn't set at all — this is why the project runs
with sensible values even before you've customized `.env`.

**Why `.env` is gitignored:** `.gitignore` in this project explicitly
excludes it — environment files often hold passwords/secrets, and
committing them to Git would leak credentials into the repo's history
permanently (even if you delete the file later, it's still in old
commits).

## 2.5 Process Management

```bash
ps aux | grep spark        # find running processes matching "spark"
kill <pid>                  # terminate a process by its process ID
command &                   # run a command in the background
```

**In this project**, running the producer and ingestion job
simultaneously requires backgrounding one of them:
```bash
python -m producer.run_producer --events-per-second 20 &
./spark_jobs/submit.sh
```
The `&` after the producer command returns control to your terminal
immediately, so you can run the next command (`submit.sh`) in the same
session instead of needing two separate terminal windows for every
combination — though the walkthroughs mostly use separate terminals for
clarity when multiple long-running processes are involved.

## 2.6 Reading Logs

```bash
docker compose logs -f airflow     # follow (stream) a service's logs live
tail -f /var/log/somefile.log       # follow a plain log file
grep "ERROR" logfile.txt             # filter lines containing "ERROR"
```

**In this project**, `Makefile`:
```makefile
logs:
	docker compose logs -f
```
`-f` ("follow") keeps the command running and streams new log lines as
they're written — essential for watching a streaming Spark job or
Airflow task in real time instead of only seeing a static snapshot.

## 2.7 `curl` for Basic HTTP Checks

`curl` sends HTTP requests from the command line — useful for checking
if a web service inside a container is actually up and responding.

**In this project**, checking Airflow's health endpoint:
```bash
curl -s http://localhost:8081/health
```
`-s` ("silent") suppresses curl's own progress output, so you only see
the actual response — used throughout the verification-gate checks to
confirm a service is reachable without opening a browser.

---

# TOPIC 3 — Git & Version Control

## 3.1 Core Commands: `init`, `add`, `commit`, `push`, `pull`

```bash
git init                          # start tracking a new repo in this folder
git add .                          # stage all changed files for the next commit
git commit -m "message"             # save a snapshot of staged changes
git push origin main                # upload local commits to a remote (e.g. GitHub)
git pull                             # download and merge remote changes
```

**In this project**, Phase 1, Day 1:
```bash
mkdir -p clickstream-lakehouse/...
cd clickstream-lakehouse
git init
```
Every phase's "Day 4/5" step ends with a commit — this project treats
**each phase as a clean, working checkpoint**:
```bash
git add .
git commit -m "Phase 1: local infra — Kafka, Postgres, Spark cluster, Airflow, all via Docker, zero cloud accounts"
```
This is a deliberate practice, not busywork: if Phase 4 breaks
something, you can inspect exactly what Phase 3 looked like before it,
using `git log` and `git diff`.

## 3.2 `.gitignore` — What Never Gets Committed

`.gitignore` lists file patterns Git should ignore entirely — they
never get staged, committed, or pushed, even with `git add .`.

**In this project**, `.gitignore`:
```
.env
__pycache__/
*.pyc
.venv/
data/bronze/
data/silver/
data/gold/
*.db
.DS_Store
airflow/logs/
airflow/airflow.db
dbt/target/
dbt/logs/
```
Each line has a reason:
- `.env` — secrets/credentials (see Topic 2.4)
- `__pycache__/`, `*.pyc` — Python's compiled bytecode cache, regenerated
  automatically, never meant to be version-controlled
- `data/bronze/`, `data/silver/`, `data/gold/` — these hold potentially
  gigabytes of generated Delta table data; committing generated data to
  Git bloats the repo forever, since Git never truly deletes history
- `*.db`, `airflow/airflow.db` — Airflow's local SQLite metadata
  database — regenerated on startup, machine-specific
- `dbt/target/`, `dbt/logs/` — dbt's compiled SQL and run logs,
  regenerated by every `dbt run`

**The general rule:** if a file is either (a) secret, or (b) can be
regenerated automatically from something else that *is* committed, it
belongs in `.gitignore`.

## 3.3 Branching Basics

A branch is an independent line of development. `main` (or `master`) is
typically the primary branch; feature branches let you work on changes
without affecting `main` until you're ready.

```bash
git checkout -b feature/add-geo-enrichment    # create and switch to a new branch
git checkout main                                # switch back to main
git merge feature/add-geo-enrichment              # bring those changes into main
```

**Why this matters even solo:** the patch phase of this project (adding
IP/geo enrichment, fixing the Silver dedup key, syncing Gold to
Postgres) is exactly the kind of change that's safer on a branch —
if the geo enrichment approach turned out to be wrong, you could
abandon that branch entirely without touching the already-working
`main`.

## 3.4 Writing Meaningful Commit Messages

A good commit message explains **why**, not just **what** (the diff
already shows *what* changed). Compare:

```
❌ "fix stuff"
❌ "update code"
✅ "Phase 4: real Bronze/Silver/Gold Delta tables — replaces Phase 3's _dev_preview parquet sinks"
✅ "patch: fix Silver dedup key to (event_id, event_timestamp) per original proposal"
```
Every commit message used across this project's phases follows the
second pattern: **what changed + why/context**, so a future reader (or
an interviewer looking at your commit history) understands the
project's evolution without needing to ask you.

## 3.5 Tagging Releases

A tag marks a specific commit as significant — typically a release
point.

```bash
git tag v1.0
git push origin main --tags
```

**In this project**, Phase 8's final step:
```bash
git add . && git commit -m "Phase 8: final docs, architecture notes, limitations, demo script"
git tag v1.0
git push origin main --tags
```
Tagging `v1.0` here specifically marks "the point where all 8 phases
plus the proposal-compliance patch are complete and verified" — useful
both for your own reference and for anyone reviewing the repo who wants
to see the project in its finished state rather than mid-development.

---

## Quick self-check before moving to Topic 4 (Docker)

You should be able to answer all of these without looking back:
- [ ] Why is `EVENT_TYPES` a tuple instead of a list in `schemas.py`?
- [ ] What does `frozen=True` actually prevent, and why does `CONFIG` use it?
- [ ] Why does the project import the Airflow DAG file by path instead of by package name in tests?
- [ ] Why is `data/bronze/` in `.gitignore` but `spark_jobs/bronze_to_silver.py` is not?
- [ ] What would happen if the `threading.Lock()` around `_sent_count` were removed?
- [ ] Why does `submit.sh` start with `set -euo pipefail`?

If any of these feel shaky, re-read that section before continuing —
Topics 4 onward assume this foundation is solid.

---
---

## Topics 4–6: Docker & Containerization · SQL · Data Engineering Core Concepts

These notes are written at interview depth — each concept includes not
just "what it is" but the vocabulary and reasoning you'd use to explain
it out loud, plus exactly where it shows up in this project's code.
Read a section, then try explaining it to yourself (or out loud) in
your own words before moving on — that's the actual test of whether
you've internalized it, not whether you can re-read it silently.

---

# TOPIC 4 — Docker & Containerization

## 4.1 What a Container Actually Is (and why it's not "a lightweight VM")

The interview-safe way to explain this: **a virtual machine virtualizes
hardware; a container virtualizes the operating system.**

A VM runs a full guest OS (its own kernel, its own drivers) on top of a
hypervisor, which itself sits on top of the host OS. That's why VMs are
measured in gigabytes and take tens of seconds to boot — you're booting
an entire operating system.

A container shares the **host machine's kernel**. It doesn't boot an OS
at all — it's a process (or group of processes) that the kernel
isolates using two Linux features:
- **Namespaces** — give the process its own isolated view of things
  like the filesystem, network interfaces, process IDs, and hostname.
  A process inside a container sees itself as PID 1, with its own
  `/etc`, even though on the host it's just an ordinary process among
  many.
- **cgroups (control groups)** — limit and account for the resources
  (CPU, memory, I/O) a process or group of processes can use.

This is why containers start in milliseconds, not seconds, and why a
container image can be tens of megabytes instead of gigabytes — you're
not shipping a kernel, just the application and its dependencies.

**In this project**, every service in `docker-compose.yml` (Kafka,
Postgres, Spark, Airflow, Metabase) is a separate container, all
sharing the same host Linux kernel, but each with its own isolated
filesystem, process tree, and (mostly) network stack.

## 4.2 Images vs. Containers vs. Volumes

This distinction trips people up in interviews if you're not precise:

- **Image** — a read-only, immutable template: a filesystem snapshot
  plus metadata (what command to run, what ports to expose). Built once
  from a `Dockerfile`, then reused to start any number of containers.
- **Container** — a running (or stopped) *instance* of an image. Has
  its own writable layer on top of the image's read-only layers. Delete
  the container, and anything written to that writable layer is gone
  — unless it was in a volume.
- **Volume** — persistent storage that exists independently of any one
  container's lifecycle. Deleting a container never deletes its
  volumes; you have to do that explicitly.

**In this project**, `docker-compose.yml`:
```yaml
volumes:
  postgres-data:
  kafka-data:
  spark-data:

services:
  postgres:
    volumes:
      - postgres-data:/var/lib/postgresql/data
```
`postgres-data` is a **named volume** — Docker manages where it
actually lives on the host disk. Every time you run
`docker compose down` (without `-v`), the `postgres-data` volume
survives, so Postgres's actual data isn't lost even though the
container itself is destroyed and recreated. Running
`docker compose down -v` explicitly destroys volumes too — which is
*exactly* why Phase 6's patch required `-v` to re-trigger Postgres's
one-time init scripts (they only run against a genuinely empty data
directory).

## 4.3 Writing a Dockerfile

A `Dockerfile` is a recipe for building an image, executed top to
bottom, each instruction adding a new layer on top of the previous one.

**In this project**, `Dockerfile.spark`:
```dockerfile
FROM bitnami/spark:3.5

USER root
RUN pip install --no-cache-dir psycopg2-binary==2.9.9
USER 1001
```
Line by line:
- `FROM bitnami/spark:3.5` — start from an existing, pre-built image
  that already has Spark installed. You almost never build a Spark
  image from a bare OS image yourself — you build *on top of* one
  someone else already got right.
- `USER root` — the base image runs as a non-root user (`1001`) by
  default, for security. Installing new system-level packages
  typically requires root privileges, so we temporarily switch.
- `RUN pip install ...` — executes a command *at build time*, baking
  its result into the image layer. This is different from a command
  that runs when the container *starts* (that's `CMD`/`ENTRYPOINT`).
- `USER 1001` — switch back to the non-root user before the image is
  finished, so containers run with the base image's original, safer
  permissions.

**Why this matters for an interview answer:** you should be able to
explain *why* you'd bake `psycopg2-binary` into a custom image instead
of installing it at container-start time every run — baking it into
the image means every container started from it already has the
dependency, instantly, with no network call or install step slowing
down (or potentially failing) every single job run.

## 4.4 Docker Compose: Services, Networks, Volumes, `depends_on`

Docker Compose is a tool for defining and running **multi-container**
applications from a single YAML file, instead of typing out long
`docker run` commands by hand for every container.

### Services
Each top-level entry under `services:` becomes one container.
**In this project**, six (later more) services: `zookeeper`, `kafka`,
`postgres`, `spark-master`, `spark-worker`, `airflow`, `metabase`.

### Networks
```yaml
networks:
  lakehouse-net:
    driver: bridge
```
A custom **bridge network** — Docker gives every container on it a
private IP and, critically, **DNS resolution by service name**. This is
why Spark can reach Kafka using the hostname `kafka` (not an IP
address) — Docker's embedded DNS resolves `kafka` to whatever
container is currently running that service.

### `depends_on` with health conditions
```yaml
kafka:
  depends_on:
    zookeeper:
      condition: service_healthy
```
Plain `depends_on` (without a condition) only guarantees **start
order**, not readiness — Kafka's container might start before
Zookeeper is actually ready to accept connections. Adding
`condition: service_healthy` makes Compose wait until Zookeeper's
`healthcheck` genuinely passes before starting Kafka — this is the
difference between "the container process has started" and "the
service inside it is actually usable."

### Healthchecks
```yaml
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
  interval: 10s
  timeout: 5s
  retries: 5
```
Docker periodically runs `test` inside the container; if it succeeds,
the container is marked `healthy`. This is what makes `depends_on`'s
`condition: service_healthy` meaningful at all.

## 4.5 Bind Mounts vs. Named Volumes

Both let a container access data outside its own writable layer, but:

- **Named volume** (`postgres-data:/var/lib/postgresql/data`) — Docker
  manages the storage location; you don't need to know or care where on
  the host it physically lives. Best for data a container *owns*
  (a database's files).
- **Bind mount** (`./spark_jobs:/opt/spark-jobs`) — maps a **specific
  host path** directly into the container. Best for code you're
  actively editing and want the container to see immediately, without
  rebuilding an image.

**In this project**:
```yaml
spark-master:
  volumes:
    - ./spark_jobs:/opt/spark-jobs   # bind mount — your code
    - ./data:/opt/lakehouse-data      # bind mount — Delta tables
    - spark-data:/bitnami              # named volume — Spark's own internals
```
Your Python job files are bind-mounted (edit on the host, Spark sees the
change instantly, no rebuild); Spark's own internal state uses a named
volume Docker manages.

## 4.6 Docker-out-of-Docker (Mounting the Docker Socket) — Advanced

This is one of the more advanced, genuinely interview-worthy things this
project does. Airflow itself runs *inside* a container, but Phase 5's
DAG needs to launch entirely **new** Spark containers to run
spark-submit jobs. How does a process inside a container start another
container?

```yaml
airflow:
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock
```
`/var/run/docker.sock` is the Unix socket the Docker **daemon**
(running on the host) listens on for API requests. Mounting it into the
Airflow container gives that container a direct line to the host's
Docker daemon — so when Airflow's `DockerOperator` "starts a container,"
it's actually asking the **host's** Docker daemon to do it, not
nesting a second Docker engine inside the first (that alternative,
genuinely running Docker-inside-Docker with its own nested daemon, is
called "Docker-in-Docker" and is a different, heavier approach).

**The gotcha this project explicitly documents:** because the *host's*
daemon creates these new containers, any **bind mount** they need must
be specified as a **host filesystem path** — a path meaningful inside
the Airflow container (like `/opt/airflow/dags`) means nothing to the
host daemon. That's why the DAG reads a `HOST_PROJECT_DIR` environment
variable (set from `${PWD}` at compose-up time) instead of constructing
mount paths relative to its own container filesystem.

**Interview framing:** "I needed Airflow to orchestrate Spark jobs
running in separate, isolated containers, so I mounted the Docker
socket to give Airflow's container access to the host's Docker API —
and I had to be careful that any volume mounts DockerOperator requested
used host paths, not container-relative paths, since the *host* daemon
is what actually creates those containers."

## 4.7 Networking Deep Dive

Every container on a custom bridge network gets:
- Its own network namespace (its own "localhost" — meaning `localhost`
  *inside* a container refers to that container, never the host or a
  sibling container)
- A hostname matching its Compose service name, resolvable via Docker's
  embedded DNS

**This project's most commonly hit networking bug**, called out
repeatedly across the walkthroughs: confusing `localhost` with a
service name.
- From your **host machine's** terminal: `localhost:9092` reaches
  Kafka (because the port is *published* — mapped from the container to
  the host).
- From **inside another container** (e.g., Spark): you must use
  `kafka:29092` — Kafka's internal listener, on Kafka's hostname within
  the Docker network. `localhost:9092` from inside Spark's container
  would mean "Spark's own container," which obviously isn't running
  Kafka.

This exact distinction is why `docker-compose.yml`'s Kafka service
defines **two listeners**:
```yaml
KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
```
One for other containers (`kafka:29092`), one for host-machine tools
like a local `kafka-console-consumer` you might run outside Docker
(`localhost:9092`).

---

# TOPIC 5 — SQL

## 5.1 Foundational Query Structure

Every SQL query, no matter how complex, is built from the same handful
of clauses, always in this logical order:
```
SELECT ... FROM ... WHERE ... GROUP BY ... HAVING ... ORDER BY ... LIMIT ...
```
Note: this is the order you **write** them in, but not the order the
database actually **evaluates** them in — `FROM`/`WHERE` are evaluated
before `SELECT`, which is why you can't reference a column alias defined
in `SELECT` inside the same query's `WHERE` clause. This is a common
interview gotcha question.

### `JOIN` types — explained by what rows survive
- **INNER JOIN** — only rows with a match on *both* sides survive.
- **LEFT JOIN** — every row from the left table survives, even with no
  match on the right (right-side columns become `NULL`).
- **RIGHT JOIN** — mirror of LEFT JOIN.
- **FULL OUTER JOIN** — every row from both sides survives regardless
  of match.

**In this project**, dbt's `relationships` test (Topic 11 territory,
but worth previewing here) is essentially checking that a LEFT JOIN
from the fact table to a dimension never produces an unmatched
(`NULL`) row — i.e., every foreign key genuinely exists in the parent
table.

## 5.2 Aggregate Functions & `GROUP BY`

`COUNT`, `SUM`, `AVG`, `MIN`, `MAX` collapse many rows into one value
**per group** defined by `GROUP BY`.

**In this project**, `dbt/models/marts/dim_users.sql`:
```sql
select
    user_id,
    min(event_timestamp) as first_seen_at,
    count(distinct session_id) as total_sessions,
    sum(revenue) as lifetime_revenue,
    count(*) filter (where event_type = 'purchase') as total_purchases
from events
group by user_id
```
This says: "for every distinct `user_id`, collapse all their event rows
into one summary row." Every non-aggregate column in `SELECT` **must**
appear in `GROUP BY`, or the database can't determine which of that
user's many rows' values to show — this is a rule every SQL engine
enforces, and a very common early SQL mistake to accidentally violate.

### `FILTER (WHERE ...)` — conditional aggregates (Postgres-specific but crucial here)
```sql
count(*) filter (where event_type = 'purchase') as total_purchases
```
This is a cleaner, more readable alternative to the older
`SUM(CASE WHEN event_type = 'purchase' THEN 1 ELSE 0 END)` pattern.
Both compute the same thing: a count restricted to rows matching a
condition, *within* a single aggregation — this is what makes it
possible to compute `view_count`, `purchase_count`, and
`add_to_cart_count` all from **one pass** over the data in
`fact_user_sessions.sql`, instead of three separate queries.

## 5.3 Window Functions

A window function computes a value **across a set of rows related to
the current row**, without collapsing those rows into one (unlike
`GROUP BY`, every original row survives).

```sql
SELECT
    user_id,
    event_timestamp,
    ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY event_timestamp) AS event_rank
FROM raw_events
```
`PARTITION BY` here means "restart the numbering for each user" —
conceptually similar to `GROUP BY`, but the rows aren't collapsed; you
still get one output row per input row, just with an extra computed
column.

**Where this project could use one (and doesn't, deliberately):** dbt's
`dim_products.sql` uses `avg(price)` via plain `GROUP BY` rather than a
window function, because it genuinely wants one summary row per
product — a good interview instinct is knowing *which* tool
(aggregate vs. window function) fits a given grain requirement.

**Where the project uses the underlying idea without literally writing
`OVER()`:** `fact_user_sessions.sql`'s
`(array_agg(product_id order by event_timestamp))[1]` achieves "first
value ordered by time, per group" — the same goal a window function
(`FIRST_VALUE(...) OVER (PARTITION BY session_id ORDER BY event_timestamp)`)
would achieve, via a different mechanism (ordered array aggregation,
then indexing the first element).

## 5.4 Upserts — `ON CONFLICT DO NOTHING` / `DO UPDATE`

An **upsert** is "insert this row, unless a row with this key already
exists, in which case do something else instead of erroring." This is
one of the most important patterns in this entire project, because it's
what makes re-running a job **safe**.

```sql
INSERT INTO raw_events (event_id, ...)
VALUES %s
ON CONFLICT (event_id) DO NOTHING
```
This says: "try to insert; if `event_id` already exists (violates the
`PRIMARY KEY` constraint), silently skip that row instead of raising an
error." **In this project**, `spark_jobs/sync_silver_to_postgres.py`
uses exactly this, which is what makes re-running the sync job after a
checkpoint reset safe — already-synced events simply get skipped, not
duplicated.

```sql
INSERT INTO gold_active_shoppers_5min (window_start, window_end, active_shoppers)
VALUES %s
ON CONFLICT (window_start, window_end) DO UPDATE SET active_shoppers = EXCLUDED.active_shoppers
```
Here, instead of skipping, the conflicting row's value is **replaced**
with the new value (`EXCLUDED` refers to the row that *would* have been
inserted). This is used for Gold tables specifically because their
numbers legitimately *change* over time (more data arrives for a
window), unlike `raw_events`, where an event's data never changes once
written.

**The interview-ready distinction:** `DO NOTHING` for immutable,
append-only facts (an event, once recorded, never changes);
`DO UPDATE` for mutable aggregates (a windowed count that gets revised
as more data arrives).

## 5.5 Primary Keys, Indexes, and Why They Matter

A `PRIMARY KEY` enforces two things simultaneously: uniqueness (no two
rows can share the value) and non-nullability. It also, in most
databases, automatically creates an index on that column.

An **index** is a separate data structure (commonly a B-tree) that lets
the database find rows matching a condition without scanning the entire
table — the difference between reading every page of a book to find one
sentence versus using the index at the back.

**In this project**, `sql/init_raw_events.sql`:
```sql
CREATE TABLE IF NOT EXISTS raw_events (
    event_id TEXT PRIMARY KEY,
    ...
);
CREATE INDEX IF NOT EXISTS idx_raw_events_session_id ON raw_events (session_id);
CREATE INDEX IF NOT EXISTS idx_raw_events_user_id ON raw_events (user_id);
```
`event_id` needs to be the primary key specifically because that's the
column the `ON CONFLICT` upsert logic keys on — you cannot write
`ON CONFLICT (event_id)` unless `event_id` has a uniqueness constraint
(primary key or unique index) backing it; Postgres will reject the
query otherwise. The extra indexes on `session_id`/`user_id` exist
because dbt's models constantly filter/group/join on those columns —
without an index, every one of those operations would require a full
table scan as `raw_events` grows.

## 5.6 CTEs — `WITH ... AS`

A Common Table Expression names a subquery so it can be referenced
later in the same query, improving readability over deeply nested
subqueries.

**In this project**, every dbt model uses exactly this pattern:
```sql
with events as (
    select * from {{ ref('stg_events') }}
)

select
    user_id,
    ...
from events
group by user_id
```
This isn't strictly necessary for a query this simple — you could
write `select ... from {{ ref('stg_events') }} group by user_id`
directly — but establishing the `with events as (...)` convention in
every model means as models grow more complex (multiple CTEs, layered
transformations), the pattern is already consistent across the whole
project.

## 5.7 Star Schema Modeling

A **star schema** organizes data into:
- **Fact tables** — the "verbs": events, transactions, measurements.
  Typically large, with numeric/measurable columns (counts, sums,
  revenue) and foreign keys pointing to dimensions.
- **Dimension tables** — the "nouns": entities you'd filter or group
  by (users, products, time). Typically smaller, with descriptive
  attributes.

The name "star" comes from the shape when diagrammed: one fact table in
the center, dimension tables radiating outward, each joined directly to
the fact table (as opposed to a **snowflake schema**, where dimensions
are further normalized into sub-dimensions, creating a more branching
shape).

**Grain** is the single most important concept to get right when
designing a fact table: it's the answer to "what does one row in this
table represent?" **In this project**, `fact_user_sessions`' grain is
explicitly **one row per session** (not one row per event) — every
column is an aggregate computed by grouping `stg_events` by
`session_id`. Getting the grain wrong (e.g., accidentally joining in a
way that produces multiple rows per session) is the single most common
star-schema bug, which is exactly why the project's verification gate
includes:
> "compare fact_user_sessions row count vs. COUNT(DISTINCT session_id)
> FROM raw_events — equal, no fan-out"

**Interview-ready one-liner:** "A star schema is built for read
performance and analyst usability — one central fact table with clear
grain, surrounded by denormalized dimensions, so a BI tool can answer
most business questions with a single join instead of navigating a
deeply normalized OLTP schema."

---

# TOPIC 6 — Data Engineering Core Concepts

This section is the theoretical backbone underneath every phase of the
project — the concepts an interviewer is really probing for when they
ask "walk me through your architecture," even if they never say these
exact terms.

## 6.1 ETL vs. ELT

- **ETL (Extract, Transform, Load)** — data is transformed *before* it
  lands in the destination system. Historically necessary when compute
  was expensive and warehouses were slow — you transformed on a
  separate, cheaper system first.
- **ELT (Extract, Load, Transform)** — raw data is loaded into the
  destination *first*, and transformation happens there, afterward,
  usually in SQL.

**In this project**, dbt is explicitly an **ELT** tool — its own
documentation describes it as "the T in ELT." dbt **never loads data
anywhere**; it only transforms data that's already landed. This is
precisely why Phase 6 needed a *separate* mechanism
(`sync_silver_to_postgres.py`, `sync_gold_to_postgres.py`) to actually
get data into Postgres before dbt could do anything with it — a gap
worth being able to explain clearly if asked "why do you have two
different Postgres-writing mechanisms instead of just using dbt for
everything?"

## 6.2 Batch vs. Streaming vs. Micro-batch

- **Batch** — process a large, bounded chunk of data at a scheduled
  interval (hourly, daily). Simple, well-understood, but introduces
  latency (data is stale until the next batch runs).
- **Streaming** — process data continuously, record by record (or in
  very small groups), as it arrives. Low latency, but more operationally
  complex (you need to handle out-of-order data, manage long-running
  processes, reason about "when is a result final").
- **Micro-batch** — a middle ground: process small, frequent batches
  (every few seconds/minutes) rather than one record at a time. Spark
  Structured Streaming is architecturally a micro-batch engine under
  the hood, even when it's conceptually presented as "streaming."

**In this project**, this exact spectrum is deliberately spanned:
Phase 3's Kafka ingestion runs continuously (true streaming, keeping up
with a live topic); Phase 5 explicitly *converts* the Silver and Gold
jobs from continuous streaming to **scheduled incremental batch runs**
(`trigger(availableNow=True)`), specifically because paying for an
always-on Spark job for a step that only needs to run every two minutes
is wasteful. Being able to articulate *why* you'd choose one over the
other for a given stage — not just that both exist — is exactly the
kind of judgment interviewers are listening for.

## 6.3 Delivery Semantics: At-Least-Once, At-Most-Once, Exactly-Once

- **At-most-once** — a message might be lost, but is never processed
  twice.
- **At-least-once** — a message is never lost, but might be processed
  more than once (e.g., after a crash and retry).
- **Exactly-once** — a message is processed precisely once, no matter
  what failures occur. Genuinely difficult to guarantee end-to-end
  across independent systems (Kafka → Spark → Delta → Postgres) without
  careful design at every hop.

**In this project's honest self-assessment** (`LIMITATIONS.md`):
> "At-least-once delivery plus idempotent dedup/upsert at each hop...
> gets close in practice, but this isn't a formally exactly-once
> pipeline."

This is the correct, mature way to describe most real-world systems —
true, formally-proven exactly-once semantics across heterogeneous
systems is rare even in large companies. What's realistic (and what
this project actually does) is: allow **at-least-once** delivery, but
make every downstream write **idempotent**, so processing the same
event twice produces the same end state as processing it once.

## 6.4 Idempotency — The Single Most Important Concept in This Project

An operation is **idempotent** if performing it multiple times produces
the same result as performing it once. This is the property that lets
you safely retry, replay, or re-run something after a failure without
fear of corrupting your data.

**Every layer of this project is deliberately made idempotent, using a
different technique per layer** — a great interview answer is being
able to name all four:
1. **Kafka producer key** (`user_id`) — guarantees per-user ordering,
   which downstream dedup logic depends on.
2. **Silver dedup** (`dropDuplicatesWithinWatermark(["event_id", "event_timestamp"])`)
   — reprocessing the same Bronze data twice doesn't create duplicate
   Silver rows.
3. **Gold MERGE upserts** — reprocessing the same window doesn't create
   duplicate Gold rows; it replaces the existing row for that window.
4. **Postgres `ON CONFLICT`** — reprocessing the same event or window
   doesn't create duplicate Postgres rows.

## 6.5 The Medallion Architecture (Bronze / Silver / Gold)

A layered data-quality architecture popularized by Databricks, where
each layer represents progressively more refined, more trustworthy
data:

- **Bronze** — raw, as-ingested, append-only. No transformation beyond
  what's needed to make the data structurally readable. Its entire
  purpose is enabling **full historical replay** — if a downstream bug
  is ever discovered, you can reprocess from Bronze without needing to
  re-fetch from the original source (which may not even be replayable,
  e.g., a Kafka topic with a retention policy that's already deleted
  old messages).
- **Silver** — cleansed, deduplicated, enriched, conformed to a
  consistent schema. This is "trustworthy row-level data," but not yet
  aggregated for any specific business question.
- **Gold** — business-level aggregates, ready for direct consumption by
  BI tools or downstream applications. Answers specific questions
  (conversion rate, active users) rather than being general-purpose.

**In this project**, this is implemented literally as three Delta
table layers, with escalating levels of processing at each hop —
Bronze does only parse/validate/quarantine; Silver adds dedup and
enrichment; Gold adds windowed aggregation. **Interview framing:**
"Bronze is deliberately 'dumb' on purpose — the value of an
immutable, unprocessed raw layer is that it's your safety net; every
transformation happens further downstream, where mistakes are cheaper
to fix and reprocess."

## 6.6 Data Quality: Schema Validation & Quarantine (Not Silent Drop)

A naive pipeline either crashes on bad data or silently drops it — both
are bad outcomes (crashing halts all processing over one bad record;
silent dropping loses data with no trace it ever happened).

**The pattern this project uses instead**, `spark_jobs/stream_ingest.py`:
records failing validation (unparseable JSON, missing required field,
unknown `event_type`) are routed to a **separate quarantine path**,
never dropped and never allowed to crash the job. This preserves an
audit trail (you can inspect exactly what was rejected and why) while
keeping the main pipeline flowing.

**Interview-ready phrasing:** "I treat data quality failures as a
routing decision, not an error-handling afterthought — bad records get
a designated place to go, so nothing disappears silently and nothing
blocks the rest of the pipeline."

## 6.7 Watermarking & Late-Arriving Data

In a streaming system, "when is a window's data complete?" is a real
question — events can arrive late (network delays, buffering, clock
skew). A **watermark** is the engine's declared threshold: "I will wait
up to X time for late data in a window before considering that window
final and releasing/aggregating it."

This is fundamentally a **trade-off between correctness and latency**:
a longer watermark captures more late data (more correct) but delays
results longer (higher latency); a shorter watermark releases results
faster but risks missing genuinely late data.

**In this project**: `withWatermark("event_timestamp", "10 minutes")`
— the engine tolerates events up to 10 minutes late before considering
a time window closed. The producer deliberately simulates late events
(~5% of traffic, up to 10 minutes late) specifically so this mechanism
has something real to prove itself against — a pipeline with perfectly
ordered data would never demonstrate whether watermarking logic
actually works.

## 6.8 Windowing (Tumbling Windows Specifically)

Grouping continuous, unbounded data into discrete time buckets so
aggregation is even possible (you can't `SUM()` a value "forever" — you
need a boundary).

- **Tumbling windows** — fixed-size, non-overlapping (e.g., every
  distinct hour: 1:00–2:00, 2:00–3:00). Each event belongs to exactly
  one window.
- **Sliding windows** — fixed-size but overlapping (e.g., a new 1-hour
  window starts every 15 minutes) — a given event can belong to
  multiple windows.
- **Session windows** — dynamic boundaries based on activity gaps
  (a window closes after N minutes of inactivity), not fixed clock
  time.

**In this project**, all three Gold aggregates use **tumbling
windows** — `window(col("event_timestamp"), "1 hour")` for
conversion rate and cart abandonment, `"5 minutes"` for active
shoppers. Each event contributes to exactly one hour's (or 5-minute
period's) statistics, never split across two.

## 6.9 Checkpointing

A checkpoint is Spark Structured Streaming's mechanism for remembering
**exactly how far it has processed** — which Kafka offsets have been
consumed, what aggregation state exists — so that if the job crashes
and restarts, it resumes from where it left off instead of either
reprocessing everything from scratch or (worse) silently skipping data.

**In this project**, every streaming/incremental job has its own,
separate checkpoint location:
```python
.option("checkpointLocation", f"{CONFIG.checkpoint_dir}/silver")
```
**A real gotcha this project's docs flag explicitly:** if the
checkpoint location isn't correctly persisted (e.g., accidentally
mounted somewhere ephemeral), every restart reprocesses the entire
topic from `earliest` again — invisible as an outright error, but
silently duplicating downstream data. This is exactly the kind of
subtle, hard-to-detect bug interviewers like hearing you've thought
about.

## 6.10 Partitioning Strategy & Data Skew

Covered briefly in Topic 1.9, worth restating with more precision here:
**partitioning is a physical data-layout decision**, chosen based on
how the data will be *queried* later.

**In this project**, Silver is written `.partitionBy("event_date")` —
because almost every downstream query (Gold aggregation, dbt models)
filters or groups by date/time. Physically separating data by date
means a query for "yesterday's events" only reads yesterday's files,
never scanning the entire table's history (this is called **partition
pruning**).

**Data skew** is the failure mode where partitioning doesn't distribute
data evenly — e.g., if one `event_date` had 100x the events of every
other date, that partition becomes a bottleneck every downstream job
has to wait on, even if every other partition finished quickly.

## 6.11 Data Contracts

A **data contract** is an explicit, enforced agreement about a
dataset's schema (and often semantics) between the producer of data and
its consumers — ideally versioned, validated automatically, and
impossible to silently break.

**This project's honest, explicitly-documented gap**: `producer/schemas.py`
and `spark_jobs/schema.py` are the *same* logical contract
(`ClickstreamEvent`'s fields) maintained by hand in **two different
languages**, with **no automated check** that they stay in sync — when
`ip_address` was added during the patch phase, it had to be manually
added to both files, and nothing would have caught it if one had been
forgotten. A real data contract system (a shared Avro/Protobuf schema
with a schema registry, validated on both the producer and consumer
side at build or runtime) would close this gap.

**Interview-ready framing:** "I have an unenforced data contract
between my producer and my Spark job — it works today because I
maintain both by hand, but I'm aware that's a real fragility, and the
production fix would be a schema registry with compatibility checking
on every deploy."

## 6.12 CDC — Change Data Capture / Change Data Feed

**CDC** broadly means: capturing *changes* to a dataset (inserts,
updates, deletes) as they happen, rather than repeatedly re-reading the
entire dataset to detect what changed. Delta Lake's specific
implementation of this idea is called **Change Data Feed (CDF)** —
enabling it on a Delta table lets you stream just the rows that
changed since you last read, instead of re-scanning the whole table.

**In this project**, Gold tables are synced to Postgres via a **full
snapshot upsert** instead of CDF — explicitly documented as a
scale-appropriate simplification (Gold tables are small, so re-reading
the whole thing every sync is cheap), with CDF named as "the natural
upgrade if these tables ever grew large enough for a full snapshot read
to be expensive." Knowing *when* a simpler approach is genuinely fine
versus when it becomes a real bottleneck — and being able to name the
more sophisticated alternative even when you didn't need it — is
exactly the kind of scaling judgment interviews probe for.

---

## Quick self-check before moving to Topic 7 (Kafka)

- [ ] Explain, without notes, the difference between a container and a
      VM, using the words "namespace" and "cgroup."
- [ ] Explain why `depends_on` alone isn't enough to guarantee a
      dependency is *ready*, not just *started*.
- [ ] Explain why `ON CONFLICT DO NOTHING` is right for `raw_events` but
      `DO UPDATE` is right for Gold tables.
- [ ] Explain "grain" in your own words, and why fan-out is the classic
      star-schema bug.
- [ ] Explain the difference between ETL and ELT, and which one dbt is.
- [ ] Explain idempotency, and name all four places this project
      enforces it.
- [ ] Explain why Bronze is "deliberately dumb."
- [ ] Explain the correctness/latency trade-off a watermark represents.

If you can answer all eight fluently, out loud, without re-reading —
you're ready for Topic 7. If any feel shaky, that's the one to revisit,
not the whole document.

---
---

## Topics 7–9: Apache Kafka · Apache Spark (Core + Structured Streaming) · Delta Lake / Lakehouse Concepts

These three topics are where the project's actual data-movement engine
lives — Kafka gets events in, Spark processes them, Delta Lake stores
the result durably. Expect these to be the sections you get asked about
most directly in a data engineering interview, so the explanations here
lean even further into "how would you say this out loud" than the
topics before.

---

# TOPIC 7 — Apache Kafka

## 7.1 What a Message Broker Is, and Why You'd Use One

Without a broker, if Service A wants to tell Service B "an event
happened," A has to call B directly — which means A now depends on B
being online, fast enough, and reachable. Add a third consumer (Service
C) and A has to call that too. This tightly couples every producer to
every consumer.

A **message broker** decouples them: producers write messages to the
broker without knowing who (if anyone) will read them; consumers read
messages without knowing who produced them. Kafka is a broker built
specifically for **high-throughput, durable, ordered** event streams —
distinguishing it from simpler message queues (like RabbitMQ, built
more around task distribution than event streaming).

**In this project**, the producer (`producer/run_producer.py`) has zero
knowledge that Spark exists at all. It just writes to a topic. Spark
(`spark_jobs/stream_ingest.py`) has zero knowledge of the producer's
internals — it just reads from that same topic. This is the entire
point: either side can be rewritten, restarted, or scaled independently.

## 7.2 Topics, Partitions, Producers, Consumers

- **Topic** — a named stream of messages (this project uses one:
  `ecommerce.clickstream`). Conceptually similar to a table name, but
  for a stream rather than a static dataset.
- **Partition** — a topic is split into one or more partitions, each an
  ordered, append-only log. **Order is only guaranteed within a single
  partition, never across partitions of the same topic.** This is the
  single most commonly misunderstood Kafka fact — people assume a topic
  is one big ordered log; it's actually N independently-ordered logs.
- **Producer** — anything writing messages to a topic.
- **Consumer** — anything reading messages from a topic.

**In this project**, the topic is created with 3 partitions:
```bash
docker exec lakehouse-kafka kafka-topics --create \
  --topic ecommerce.clickstream \
  --bootstrap-server localhost:9092 \
  --partitions 3 --replication-factor 1
```
Three partitions means up to three consumers (or three Spark tasks) can
read this topic **in parallel**, each handling a different partition —
this is Kafka's core mechanism for horizontal scalability.

## 7.3 Producing/Consuming via the CLI

```bash
# produce (reads lines from stdin, publishes each as a message)
docker exec -i lakehouse-kafka kafka-console-producer \
  --topic ecommerce.clickstream --bootstrap-server localhost:9092

# consume (prints messages as they arrive)
docker exec lakehouse-kafka kafka-console-consumer \
  --topic ecommerce.clickstream --bootstrap-server localhost:9092 \
  --from-beginning --max-messages 5
```
`--from-beginning` tells the consumer to start reading from the
**earliest** available offset in the topic, rather than only new
messages published after the consumer connects — essential for
verifying data that was produced *before* you started watching.

## 7.4 Partition Keys and Ordering Guarantees

When you produce a message with a **key**, Kafka hashes that key to
deterministically choose a partition — the same key always lands on the
same partition (as long as the partition count doesn't change).

**In this project**, `producer/kafka_producer.py`:
```python
future = self._producer.send(
    CONFIG.topic,
    key=event.user_id,
    value=event.to_json(),
)
```
Keying by `user_id` guarantees **all of one user's events land on the
same partition, in the order they were produced**. This isn't a nice-to-
have — it's a hard requirement for Silver's dedup logic and any
session-based processing downstream: if a user's events were scattered
randomly across partitions, you could never rely on processing them in
the order they actually happened.

**Interview framing:** "I key my Kafka messages by user ID specifically
to get a partition-level ordering guarantee for each user's event
stream — without that, my downstream deduplication and session logic
would be operating on data that could arrive out of order relative to
itself."

## 7.5 Consumer Groups & Offsets

An **offset** is a message's position within its partition (0, 1, 2,
...) — Kafka's way of tracking "how far has this consumer read."

A **consumer group** is a named set of consumers that split a topic's
partitions among themselves — each partition is read by exactly *one*
consumer within a group at a time, letting a group process a topic in
parallel without any two members reading the same message. (Spark's
Kafka connector manages consumer group membership internally, based on
the checkpoint's tracked offsets, rather than exposing it directly to
your job code.)

`--from-beginning` (earliest) vs. the default (latest) is really a
choice about **offset starting position** — this project deliberately
uses:
```python
.option("startingOffsets", "earliest")
```
in `spark_jobs/stream_ingest.py`, so that the very first time the job
runs, it processes everything already sitting in the topic rather than
only events produced after the job starts.

## 7.6 Replication Factor & `acks`

**Replication factor** is how many copies of each partition Kafka
maintains across different brokers — for fault tolerance (if one broker
dies, a replica elsewhere still has the data). This project uses
`--replication-factor 1` — meaning **zero fault tolerance** at the
Kafka layer, a deliberate simplification appropriate for a single-broker
local setup (a real production cluster would use at least 3).

**`acks`** controls how many replicas must confirm receipt before the
producer considers a send successful:
```python
KafkaProducer(
    ...,
    acks="all",
    retries=5,
)
```
`acks="all"` (sometimes written `acks=-1`) means "wait for **all** the
partition's replicas to acknowledge," the strongest durability
guarantee Kafka offers — trading a small amount of latency for
confidence that a message isn't lost even if the leader replica
immediately fails. With `replication_factor=1` there's only one replica
to begin with, so `acks="all"` here is really about matching the
*intent* of a production-grade config, ready to actually matter the
moment replication factor is increased.

## 7.7 Zookeeper's Role (Pre-KRaft Kafka)

Historically, Kafka depended on **Zookeeper** — a separate distributed
coordination service — to manage cluster metadata: which broker is the
leader for each partition, cluster membership, configuration. (Newer
Kafka versions support **KRaft** mode, which removes the Zookeeper
dependency by having Kafka brokers manage this consensus themselves —
worth knowing this transition exists, even though this project uses the
traditional Zookeeper-dependent setup for broader compatibility with
well-documented images.)

**In this project**, `docker-compose.yml`'s Kafka service explicitly
waits on Zookeeper's healthcheck before starting:
```yaml
depends_on:
  zookeeper:
    condition: service_healthy
```
Without Zookeeper up and healthy first, Kafka has nowhere to register
itself as a broker and will fail to start correctly.

## 7.8 Backpressure & Batching (`linger.ms`)

**Backpressure** is what happens when a producer generates messages
faster than the broker (or downstream consumer) can absorb them — if
unmanaged, this can overwhelm a system.

`linger.ms` is a producer-side batching knob: instead of sending every
single message immediately, the producer waits up to this many
milliseconds to accumulate a bigger batch, trading a small amount of
per-message latency for significantly better network/throughput
efficiency (fewer, larger requests instead of many tiny ones).

```python
KafkaProducer(..., linger_ms=20)
```
**Interview framing:** "I tuned `linger.ms` to batch outgoing messages
slightly rather than sending each one immediately — a small latency
cost per message, but meaningfully better throughput, which matters
more at the event volumes a real production system would see."

## 7.9 Why Partition Count Limits Downstream Parallelism

This is a subtle but important scaling fact: **you cannot have more
parallel consumers actively reading a topic than it has partitions.**
If this project's topic had only 1 partition, no matter how many Spark
executors you throw at ingestion, only one of them could ever be
reading at a time — the other executors would sit idle. This is exactly
why the topic is created with 3 partitions up front, even at small
scale: partition count is much harder to increase later without
disrupting existing key-based ordering guarantees than it is to set
appropriately from the start.

---

# TOPIC 8 — Apache Spark (Core + Structured Streaming)

## 8.1 The Cluster Model: Driver, Master, Worker/Executor

- **Driver** — the process running your actual application code (the
  `SparkSession`, your transformations). It builds the execution plan
  and coordinates everything else, but doesn't do the heavy data
  processing itself.
- **Master** — (in Spark's standalone cluster mode, used in this
  project) the process that tracks which workers are available and
  allocates resources to applications.
- **Worker** — a machine (or container) that runs one or more
  **executors** — the processes that actually execute tasks and hold
  data in memory.

**In this project**, `docker-compose.yml`:
```yaml
spark-master:
  environment:
    SPARK_MODE: master
spark-worker:
  environment:
    SPARK_MODE: worker
    SPARK_MASTER_URL: spark://spark-master:7077
```
When you run `spark-submit --master spark://spark-master:7077 ...`,
your script becomes the **driver**, connecting to the master, which
allocates the single available worker's resources to run your job's
tasks.

## 8.2 DataFrame API Basics

A Spark DataFrame is a distributed, table-like collection of data
(conceptually similar to a pandas DataFrame or a SQL table), but
partitioned across a cluster and evaluated **lazily** — transformations
like `.select()`, `.filter()`, `.withColumn()`, `.groupBy()` build up a
plan; nothing actually executes until an **action** (like `.count()`,
`.collect()`, or starting a streaming query) triggers it.

**In this project**, `spark_jobs/stream_ingest.py`:
```python
parsed = (
    raw.selectExpr("CAST(value AS STRING) AS json_str", "timestamp AS kafka_ingest_ts")
    .withColumn("event", from_json(col("json_str"), schema))
)
```
`selectExpr` lets you use raw SQL-style expressions inline (`CAST(...)`)
instead of composing them from individual function calls — useful for
quick type casts and column renames like this.

## 8.3 Structured Streaming: `readStream`/`writeStream`, Output Modes

Structured Streaming treats a stream as **an unbounded, continuously
appending table** — the same DataFrame API you'd use for batch data
works for streaming data, just started with `.readStream` instead of
`.read`, and finished with `.writeStream.start()` instead of `.write.save()`.

**Output modes** describe what gets written on each micro-batch:
- **Append** — only *new* rows since the last trigger are written.
  Cannot be used with aggregations that might later be revised.
- **Update** — only rows that *changed* since the last trigger are
  written (existing unchanged rows are skipped). Used for streaming
  aggregations, since a window's total can be revised as more data
  arrives.
- **Complete** — the *entire* result table is rewritten every trigger.
  Rarely practical at scale.

**In this project**: Bronze/Silver writes use `outputMode("append")`
(new events only, never revised); Gold's windowed aggregates use
`outputMode("update")` — because a hard truth of streaming aggregation
is that a window's count for "the last hour" can keep changing as more
(possibly late) data for that hour arrives, so `append` would be
outright incorrect here — it would produce a new row every time,
instead of revising the existing one.

## 8.4 `withWatermark()` — Syntax and Semantics

```python
valid_df = (
    parsed_events
    .withColumn("event_timestamp", to_timestamp(col("event_timestamp")))
    .withWatermark("event_timestamp", "10 minutes")
)
```
This declares: "track the maximum `event_timestamp` seen so far; once
you've seen a timestamp T, consider any event older than T minus 10
minutes as too late to affect an already-finalized window." This is
what lets Spark eventually **discard old streaming state** (it doesn't
need to remember data older than the watermark forever) while still
tolerating genuinely late-but-not-too-late data.

## 8.5 `from_json`, Schema Enforcement, Handling Malformed Records

`from_json(col, schema)` parses a JSON string column into a structured
column matching the given schema. Critically, **if a string doesn't
match the schema, `from_json` doesn't throw an exception — it returns
`null`** for that row's parsed column. This is exactly the behavior
`parse_and_validate()` in `spark_jobs/stream_ingest.py` depends on:
```python
is_parseable = col("event").isNotNull()
```
A `null` in the `event` column after `from_json` is the signal that
this particular row failed to parse — checked explicitly, rather than
relying on an exception (which would crash the entire streaming query
over a single bad message).

## 8.6 Triggers: `processingTime` vs. `availableNow`

A **trigger** controls *when* Spark Structured Streaming processes a
new micro-batch:
- **`processingTime="X seconds"`** — process a new micro-batch every X
  seconds, for as long as the query runs. This is what makes a job
  "continuous" — it never stops on its own.
- **`availableNow=True`** — process **everything currently available**,
  across as many micro-batches as needed, then **stop** automatically
  once caught up. This is what makes a Structured Streaming job
  schedulable by an external orchestrator (like Airflow) as a periodic
  batch task, instead of needing to run forever as a supervised process.

**In this project**, Phase 5's entire architectural pivot is built on
this one distinction:
```python
if trigger_mode == "continuous":
    query = writer.trigger(processingTime="15 seconds").start(...)
else:
    query = writer.trigger(availableNow=True).start(...)
```
Kafka ingestion stays `continuous` (has to keep up with a live topic);
Silver/Gold/Postgres-sync switch to `availableNow`, letting Airflow
schedule them every 2 minutes instead of running an idle Spark process
between useful work.

## 8.7 `dropDuplicatesWithinWatermark`

A streaming-only deduplication method: it drops rows with a
previously-seen key, but — critically — only needs to remember keys
seen **within the current watermark window**, not the table's entire
history. This bounds memory usage on a long-running job; without the
watermark bound, Spark would have to remember every key it had ever
seen, forever, eventually exhausting memory.

```python
deduped = bronze.dropDuplicatesWithinWatermark(["event_id", "event_timestamp"])
```
**Why this errors on a static (batch) DataFrame:** the method's entire
premise is "expire old keys once they fall outside the watermark" —
without a streaming context and an active watermark, that premise
doesn't apply, so Spark rejects the call outright rather than silently
behaving differently. This is exactly why this project's tests for
`enrich()` (the batch-safe half of the transform) had to be split apart
from the dedup logic itself, which can only be verified via an actual
end-to-end streaming run.

## 8.8 `foreachBatch` — Dropping to Batch Semantics Inside a Streaming Job

`foreachBatch` gives you the underlying **micro-batch as a plain,
static DataFrame**, letting you use APIs that only exist in batch mode
(like Delta's `MERGE`, or writing to Postgres via `psycopg2`) *inside*
an otherwise-streaming query.

```python
def upsert_batch(batch_df: DataFrame, batch_id: int) -> None:
    batch_df.select(*COLUMNS).foreachPartition(_write_partition)

writer = silver.writeStream.foreachBatch(upsert_batch)
```
**In this project**, this is the mechanism behind both Gold's MERGE
upserts and the Postgres sync jobs — anywhere the project needs
"streaming cadence, but batch-style write logic," `foreachBatch` is the
bridge between the two.

## 8.9 Windowed Aggregations

```python
silver.groupBy(window(col("event_timestamp"), "1 hour")).agg(...)
```
`window()` is a Spark SQL function that buckets a timestamp column into
fixed-size time windows, producing a struct column (`window.start`,
`window.end`) you then `groupBy` like any other column. Combined with
`withWatermark`, Spark knows when a given window can be considered
"final enough" to emit, versus still open and awaiting more (possibly
late) data.

## 8.10 User-Defined Functions (UDFs)

A UDF wraps ordinary Python logic so it can run as part of a Spark
transformation, row by row, across a distributed dataset.

```python
_region_udf = udf(ip_to_region, SparkStringType())
df.withColumn("region", _region_udf(col("ip_address")))
```
**The performance cost to be aware of (a genuine interview topic):**
native Spark SQL functions (`when`, `col`, `to_date`, etc.) execute
inside the JVM, often vectorized. A Python UDF requires **serializing
each row's data out of the JVM, into a Python process, running your
Python code, then serializing the result back** — meaningfully slower
per-row than a native function. It's the right tool when the logic
genuinely can't be expressed with built-in functions (as with
`ip_to_region`'s custom bucketing logic here), but not a default choice
for anything a native function could do instead.

## 8.11 Shuffle Partitions & Data Skew

A **shuffle** happens whenever Spark needs to redistribute data across
the cluster to satisfy an operation — most commonly `groupBy` or a
join on non-co-located keys. `spark.sql.shuffle.partitions` controls
how many partitions the *result* of a shuffle is split into.

```python
.config("spark.sql.shuffle.partitions", CONFIG.shuffle_partitions)
```
Set too low, and each partition after a shuffle holds too much data
(memory pressure, slow tasks). Set too high on a small cluster (like
this project's single worker), and you pay overhead scheduling many
tiny tasks for very little actual data each. This project's default of
4 is deliberately modest, matched to a single-worker local cluster —
the original proposal specifically called out tuning this value to
avoid data skew as an "industrial problem-solving" requirement.

## 8.12 Checkpoint Directories — What's Actually Stored There

A checkpoint directory holds everything Structured Streaming needs to
resume correctly after a restart: the Kafka offsets already processed,
any stateful aggregation data (like in-progress window counts), and
metadata about the query's configuration. Each streaming query needs
its **own** checkpoint directory — sharing one between two different
queries corrupts both.

```python
.option("checkpointLocation", f"{CONFIG.checkpoint_dir}/bronze_valid")
.option("checkpointLocation", f"{CONFIG.checkpoint_dir}/bronze_quarantine")
```
Note the two separate subpaths for the valid and quarantine sinks in
`stream_ingest.py` — even though they originate from the same source
stream, they're two independent queries, each needing independent
checkpoint state.

---

# TOPIC 9 — Delta Lake / Lakehouse Concepts

## 9.1 What Delta Lake Adds Over Plain Parquet

Parquet is just a columnar file format — efficient to store and query,
but with **no transactional guarantees**. If a job crashes halfway
through writing Parquet files, you can be left with a partially-written,
inconsistent dataset, and there's no built-in way to know which files
are "committed" versus "in progress."

Delta Lake adds a **transaction log** on top of Parquet files, giving
you:
- **ACID transactions** — a write either fully succeeds or fully fails;
  readers never see a half-written state.
- **Schema enforcement** — writes that don't match the table's expected
  schema are rejected by default (rather than silently corrupting the
  table).
- **Time travel** — the ability to query the table as it existed at a
  previous version.
- **Unified batch and streaming** — the same table can be a streaming
  source and a batch-query target simultaneously.

**Interview framing:** "Delta Lake is essentially 'Parquet plus a
transaction log' — it gives a data lake the transactional guarantees
you'd normally only get from a traditional database, without giving up
the scalability and openness of files-on-object-storage."

## 9.2 The `_delta_log` Directory

Every Delta table has a hidden `_delta_log/` subdirectory containing
JSON (and periodically, consolidated Parquet "checkpoint") files, each
representing one committed transaction against the table — which files
were added, which were removed, what the schema was at that point.
**This log *is* the table's source of truth** — the actual data files
are meaningless without it (a reader has to consult the log to know
which Parquet files are currently valid).

**In this project**, this exact mechanic is what powers Airflow's
freshness gate (`airflow/dags/clickstream_pipeline.py`):
```python
commit_files = glob.glob(os.path.join(log_dir, "*.json"))
latest_mtime = max(os.path.getmtime(f) for f in commit_files)
```
Instead of needing a running Spark session just to check "has this
table received new data recently," the DAG reads the transaction log's
own file timestamps directly off disk — a lightweight, Spark-free way
to answer that question, precisely because the log's presence and
recency *is* the signal.

## 9.3 `MERGE INTO` Semantics

`MERGE` is SQL (and DataFrame API) syntax for **conditional upserts**:
for each row in a source dataset, either update a matching existing row
in the target table, insert it as new, or (optionally) delete a
matching target row — all as one atomic operation.

```python
(
    target.alias("target")
    .merge(batch_df.alias("source"), "target.window_start = source.window_start AND target.window_end = source.window_end")
    .whenMatchedUpdateAll()
    .whenNotMatchedInsertAll()
    .execute()
)
```
**In this project**, this is exactly how Gold tables stay correct as
streaming aggregates get revised: a window's row is *updated in place*
if it already exists (more data arrived for that window), or *inserted*
if this is the first time that window has been seen. Without `MERGE`,
you'd need to manually delete-then-reinsert or accept duplicate rows —
`MERGE` expresses "upsert" as a single, atomic, declarative operation.

## 9.4 Reading/Writing Delta Tables: Batch vs. Streaming Mode

```python
# batch write
df.write.format("delta").mode("overwrite").save(path)

# batch read
spark.read.format("delta").load(path)

# streaming write
df.writeStream.format("delta").outputMode("append").start(path)

# streaming read
spark.readStream.format("delta").load(path)
```
The same underlying table supports all four — this is what "unified
batch and streaming" means concretely. **In this project**, Bronze and
Silver are *written* via streaming (`writeStream`), but Gold's sync job
*reads* them via streaming too (`bronze_to_silver.py` reads Bronze with
`readStream`), while the Gold→Postgres sync job reads Gold via plain
**batch** `spark.read` — a deliberate choice (see 9.6) rather than an
oversight.

## 9.5 Time Travel

Delta tables can be queried as of a previous version or timestamp:
```python
spark.read.format("delta").option("versionAsOf", 3).load(path)
spark.read.format("delta").option("timestampAsOf", "2026-01-01").load(path)
```
This works because the transaction log never deletes old file
references outright (subject to a retention period) — "going back in
time" is really just asking the log "what did the valid file set look
like at version N?" This project doesn't use time travel operationally,
but it's worth knowing it's available — e.g., for debugging "what did
Gold look like right before this dashboard number looked wrong?"
without needing a separate backup system.

## 9.6 Change Data Feed (CDF) vs. Full-Snapshot Sync

Change Data Feed is Delta's built-in mechanism for **streaming only the
rows that changed** since a given version, rather than re-reading an
entire table — enabled per-table (`delta.enableChangeDataFeed = true`)
and consumed via `.option("readChangeFeed", "true")`.

**In this project's own documented trade-off**: Gold→Postgres syncing
uses a **full-snapshot batch read + upsert** instead of CDF, explicitly
because Gold's tables are small (one row per time window) — re-reading
the whole table every sync is cheap at this scale. `LIMITATIONS.md`
names CDF as "the natural upgrade if these tables ever grew large
enough for a full snapshot read to be expensive." This is a great
example of the kind of judgment call worth narrating explicitly in an
interview: **choosing the simpler mechanism deliberately, with a named
condition for when you'd switch to the more sophisticated one** — not
just defaulting to the simple option out of not knowing the advanced
one exists.

---

## Quick self-check before moving to Topic 10 (Airflow)

- [ ] Explain why ordering is only guaranteed *within* a Kafka partition,
      never across partitions of the same topic.
- [ ] Explain why this project keys its Kafka messages by `user_id`.
- [ ] Explain the difference between `outputMode("append")` and
      `outputMode("update")`, and why Gold specifically needs the latter.
- [ ] Explain why `dropDuplicatesWithinWatermark` only works on a
      streaming DataFrame, never a static one.
- [ ] Explain what `trigger(availableNow=True)` actually changes about
      how a Spark job behaves, and why that matters for orchestration.
- [ ] Explain, in one sentence, what Delta Lake adds on top of plain
      Parquet.
- [ ] Explain why the Airflow freshness gate reads `_delta_log` directly
      instead of starting a Spark session.
- [ ] Explain why Gold syncs to Postgres via full snapshot instead of
      Change Data Feed, and what would make you switch.

If you can answer all eight fluently, you're ready for Topic 10. These
three topics (Kafka, Spark, Delta) are the technical core of the whole
project — worth over-preparing on relative to the others.

---
---

## Topics 10–12: Apache Airflow · dbt (data build tool) · PostgreSQL as a Warehouse

These three topics cover the "glue and modeling" layer of the project —
Airflow decides *when* things run, dbt decides *what shape* the final
data takes, and Postgres is the place both of those things ultimately
converge. This is also the layer most likely to come up if an
interviewer asks about orchestration or analytics engineering
specifically, rather than the lower-level streaming internals.

---

# TOPIC 10 — Apache Airflow

## 10.1 DAGs, Tasks, and Operators — the Core Mental Model

A **DAG** (Directed Acyclic Graph) is Airflow's representation of a
workflow: a set of steps (**tasks**) with dependencies between them,
guaranteed to have no cycles (a task can never depend, even indirectly,
on itself). "Directed" means dependencies have a direction (A must run
before B); "acyclic" means you can't loop back.

An **operator** is a template for a single task — it defines *what kind
of work* a task does (run Python code, run a Docker container, run a
SQL query, wait for a condition). You don't write "a task"; you
instantiate an operator, and that instance becomes a task in the DAG.

**In this project**, `airflow/dags/clickstream_pipeline.py` defines one
DAG with six tasks, each an instance of either `PythonOperator` or
`DockerOperator`:
```python
with DAG(dag_id="clickstream_silver_gold_pipeline", ...) as dag:
    freshness_gate = PythonOperator(task_id="check_bronze_freshness", ...)
    run_bronze_to_silver = DockerOperator(task_id="run_bronze_to_silver", ...)
    ...
```

## 10.2 `PythonOperator` and Task Dependencies (`>>`)

`PythonOperator` runs an ordinary Python function as a task:
```python
freshness_gate = PythonOperator(
    task_id="check_bronze_freshness",
    python_callable=check_bronze_freshness,
)
```
The `>>` operator declares "runs after" — it's operator overloading in
Python (the same `>>` that normally means bit-shift is redefined by
Airflow's classes to mean "set this as a downstream dependency").
```python
freshness_gate >> run_bronze_to_silver >> run_silver_to_gold
run_silver_to_gold >> sync_silver_to_postgres >> sync_gold_to_postgres >> run_dbt
```
This reads naturally left to right: the freshness check must succeed
before Bronze→Silver runs, which must succeed before Silver→Gold, and
so on — a straight-line chain, in this project's case, though `>>` can
express branching (`task_a >> [task_b, task_c]`, fan-out to two
parallel tasks) just as easily.

## 10.3 Schedule Intervals, `catchup`, `start_date`

```python
with DAG(
    ...,
    schedule=timedelta(minutes=2),
    start_date=datetime(2026, 1, 1),
    catchup=False,
    ...
)
```
- **`schedule`** — how often the DAG should run (here: every 2 minutes).
- **`start_date`** — the earliest date Airflow considers this DAG
  "active" from. A DAG doesn't run before its `start_date`, even if
  Airflow itself started earlier.
- **`catchup`** — if `True` (the confusing default), Airflow will try
  to "catch up" by running the DAG once for *every* scheduled interval
  that has passed since `start_date` — which, for a DAG scheduled every
  2 minutes with a `start_date` months in the past, would mean tens of
  thousands of backfill runs firing immediately. **`catchup=False`**
  disables this — the DAG only runs going forward from whenever it's
  unpaused, which is almost always what you actually want for a
  continuously-running operational pipeline (as opposed to a
  historical backfill job, where `catchup=True` is genuinely useful).

## 10.4 `DockerOperator` — Running Containerized Tasks

`DockerOperator` launches a **new, separate container** to execute a
task's work, rather than running Python code inline in Airflow's own
process.

```python
run_bronze_to_silver = DockerOperator(
    task_id="run_bronze_to_silver",
    image=SPARK_IMAGE,
    command=_spark_submit_command("bronze_to_silver.py"),
    network_mode=LAKEHOUSE_NETWORK,
    mounts=[SPARK_JOBS_MOUNT, DATA_MOUNT],
    auto_remove="success",
    docker_url="unix://var/run/docker.sock",
)
```
- **`image`** — which Docker image to run the task in (here, the
  custom Spark image with `psycopg2` baked in).
- **`network_mode`** — which Docker network to attach the new container
  to, so it can resolve service names like `spark-master` and `kafka`.
- **`mounts`** — bind mounts, using **host** paths (see Topic 4.6 for
  why).
- **`auto_remove="success"`** — automatically delete the container once
  it finishes successfully, so completed task-containers don't pile up.
- **`docker_url`** — where to find the Docker daemon to talk to (the
  socket mounted into Airflow's own container).

**Why use this instead of just running Spark code directly inside
Airflow's container?** Isolation and resource visibility — each task
gets its own container, with its own resource limits, visible
separately in `docker stats`/`docker ps`, rather than everything
competing inside Airflow's single process.

## 10.5 Retries, `retry_delay`, `execution_timeout`

```python
default_args = {
    "owner": "clickstream-lakehouse",
    "retries": 2,
    "retry_delay": timedelta(seconds=30),
    "execution_timeout": timedelta(minutes=5),
}
```
- **`retries`** — how many times Airflow automatically retries a failed
  task before marking it permanently failed. Transient failures (a
  container briefly failing to pull, a momentary network blip) are
  common enough in real systems that automatic retry is standard
  practice, not a crutch.
- **`retry_delay`** — how long to wait between retry attempts.
- **`execution_timeout`** — the maximum time a single task run is
  allowed before Airflow forcibly kills it and marks it failed — a
  safety net against a task hanging forever (e.g., a Spark job stuck
  waiting on a resource that will never become available).

## 10.6 `max_active_runs` — Preventing Overlapping Runs

```python
max_active_runs=1
```
This guarantees **at most one run of this DAG is active at a time** —
if a run scheduled for 10:00 is still executing at 10:02 (the next
scheduled time), Airflow will not start a second, overlapping run. This
matters specifically because two concurrent runs of this DAG would both
be reading/writing the *same* incremental window of Bronze/Silver/Gold
data — allowing that to happen could produce race conditions or
duplicate processing, which is exactly the class of bug idempotency
(Topic 6.4) is meant to prevent, but preventing the overlapping run in
the first place is simpler than relying on downstream idempotency alone.

## 10.7 Freshness-Gate / Sensor Patterns

A **sensor** (a specialized operator type) waits for a condition to
become true before letting downstream tasks proceed — classically
"wait until a file appears" or "wait until an API returns success."

**In this project**, the freshness gate is a simpler, related pattern —
not a true Airflow `Sensor` (which polls repeatedly), but a
`PythonOperator` that **fails outright** if a condition isn't met,
rather than retrying/polling:
```python
def check_bronze_freshness(**_context) -> None:
    assert_bronze_is_fresh(BRONZE_VALID_PATH, MAX_STALENESS_SECONDS)
```
**Interview framing:** "Before running Silver and Gold transforms, I
gate the DAG on a freshness check against Bronze's Delta transaction
log — if ingestion has silently stopped, I want the DAG to fail loudly
and immediately, not waste compute reprocessing a table that received
no new data."

## 10.8 Executors: `SequentialExecutor` vs. `LocalExecutor` vs. `CeleryExecutor`

An **executor** determines *how* Airflow actually runs tasks:
- **`SequentialExecutor`** — runs one task at a time, in a single
  process. Simplest, but no parallelism at all — used in this project
  (`standalone` mode) purely for local-dev simplicity.
- **`LocalExecutor`** — runs multiple tasks in parallel using multiple
  processes on the same machine Airflow itself runs on.
- **`CeleryExecutor`** (and similar distributed executors) — distributes
  task execution across a fleet of separate worker machines, for real
  production scale.

**Interview framing:** "I used `SequentialExecutor` deliberately for
this local, single-machine project — it's the simplest correct choice
here since I only have one worker node anyway. A production deployment
running many DAGs with real parallelism requirements would need
`LocalExecutor` at minimum, or `CeleryExecutor`/`KubernetesExecutor` for
distributed scale."

## 10.9 The `airflow/` Package-Name Collision (Advanced, Project-Specific)

Covered fully in Topic 1.3, worth restating here in Airflow-specific
terms: because this repo has a **folder** named `airflow/` at its root,
and the **installed pip package** is *also* named `airflow`, a plain
`import airflow.dags.clickstream_pipeline` is ambiguous — Python's
import system might resolve `airflow` to the local folder rather than
the real library, breaking every `from airflow import DAG`-style import
inside that very file.

The project's fix: load the DAG file **by file path** using
`importlib.util.spec_from_file_location(...)` in tests, rather than by
package import — sidestepping the ambiguity entirely, since file-path
loading never has to decide between two things named `airflow`.

## 10.10 Docker Socket Mounting — Security Implications

Covered mechanically in Topic 4.6; worth being explicit about the
**security** angle here, since it's a natural interview follow-up.
Mounting `/var/run/docker.sock` into a container effectively gives that
container **root-equivalent control over the entire host** — anything
with access to the Docker socket can launch a privileged container that
mounts the host's filesystem, which is as powerful as root access on
the host itself. This is why `docker-compose.yml` running Airflow as
`user: "0:0"` (root) is explicitly flagged in `LIMITATIONS.md` as a
local-dev trade-off, not a production-acceptable pattern — a real
deployment would either avoid DockerOperator's socket-mounting approach
entirely (favoring the `KubernetesPodOperator` in a real cluster, which
doesn't require this) or tightly restrict which processes/users can
reach the socket.

---

# TOPIC 11 — dbt (data build tool)

## 11.1 What dbt Actually Does (and Doesn't Do)

dbt's entire job is: **take SQL `SELECT` statements you write, and
manage running them against your warehouse as tables/views, in the
correct dependency order, with tests.** It has no concept of "loading"
data from an external source — that's explicitly out of scope (see
Topic 6.1's ETL vs. ELT discussion). It's often summarized as "dbt lets
analytics engineers apply software engineering practices — version
control, testing, modularity — to SQL transformations."

## 11.2 `source()` vs. `ref()` — the Two Ways to Reference Data

- **`source()`** — references a raw table that dbt did **not** create;
  something external (in this project, `raw_events`, populated by
  `sync_silver_to_postgres.py`, not by dbt itself).
- **`ref()`** — references another dbt **model** (a table/view dbt
  itself creates from a `.sql` file).

**In this project**, `dbt/models/staging/sources.yml`:
```yaml
sources:
  - name: raw
    schema: public
    tables:
      - name: raw_events
```
and `stg_events.sql`:
```sql
with source as (
    select * from {{ source('raw', 'raw_events') }}
)
select event_id, event_type, ... from source
```
Then every mart model (`dim_users.sql`, `fact_user_sessions.sql`, etc.)
uses `{{ ref('stg_events') }}` — **never** referencing `raw_events`
directly. This isn't a stylistic preference: it's what lets dbt build a
**dependency graph** (its "DAG," conceptually similar to Airflow's, but
for SQL models) — dbt knows `fact_user_sessions` depends on
`stg_events`, which depends on the `raw_events` source, purely by
parsing these `ref()`/`source()` calls, and uses that graph to decide
build order and to draw the lineage diagram `dbt docs serve` shows.

## 11.3 Materializations: `view` vs. `table`

A **materialization** determines how dbt physically implements a model
in the warehouse:
- **`view`** — a saved query; every time it's queried, the underlying
  SQL re-runs live against the source data. Always up to date, but
  potentially slower for expensive queries queried often.
- **`table`** — the query's result is physically stored; queries against
  it read the stored data, not re-running the original SQL. Faster to
  query, but only as fresh as the last `dbt run`.

**In this project**, `dbt_project.yml`:
```yaml
models:
  clickstream_lakehouse:
    staging:
      +materialized: view
    marts:
      +materialized: table
```
Staging models are cheap, thin pass-throughs — a view is fine, and
staying a view means they're always current with zero extra storage.
Mart models (the fact/dimension tables) involve real aggregation work
(`GROUP BY`, window-like array aggregation) that you don't want
recomputed on every single query a BI tool sends — materializing them
as tables means that aggregation work happens once, at `dbt run` time,
not once per dashboard click.

## 11.4 Built-in Tests: `unique`, `not_null`, `relationships`

dbt ships generic, reusable tests you attach to columns in YAML, rather
than hand-writing SQL assertions:
```yaml
columns:
  - name: session_id
    tests: [unique, not_null]
  - name: user_id
    tests:
      - not_null
      - relationships:
          to: ref('dim_users')
          field: user_id
```
- **`unique`** — compiles to a query checking for duplicate values in
  this column; fails if any exist.
- **`not_null`** — checks for `NULL` values.
- **`relationships`** — dbt's version of a referential integrity check:
  every value in this column must exist somewhere in the referenced
  model's specified column (conceptually, "would a `LEFT JOIN` to that
  table ever produce an unmatched row for this column?").

Running `dbt test` executes every defined test as its own generated SQL
query and reports pass/fail — **this project's exact mechanism** for
catching the "fan-out" star-schema bug (Topic 5.7) automatically,
rather than relying on manually eyeballing row counts every time.

## 11.5 `dbt run`, `dbt test`, `dbt docs generate`/`serve`

- **`dbt run`** — executes every model's `SELECT`, materializing each as
  configured, in dependency order.
- **`dbt test`** — executes every defined test, reporting pass/fail per
  test.
- **`dbt docs generate`** — builds a static documentation site,
  including an interactive **lineage graph** showing every model's
  upstream/downstream dependencies (derived from the same `ref()`/
  `source()` calls discussed in 11.2).
- **`dbt docs serve`** — serves that generated documentation locally in
  a browser.

**In this project's demo script**, showing `dbt docs serve`'s lineage
graph (`stg_events → fact_user_sessions / dim_users / dim_products /
dim_time`) is called out as a specific, visual way to demonstrate the
project's data modeling to someone without them reading a single line
of SQL.

## 11.6 `profiles.yml` vs. `dbt_project.yml`

- **`dbt_project.yml`** — project-level configuration: model paths,
  materialization defaults, which profile to use. Committed to the
  repo.
- **`profiles.yml`** — **connection credentials** (host, user, password,
  database) for the actual warehouse dbt connects to. Normally lives
  outside the project (in `~/.dbt/`) specifically so credentials never
  accidentally get committed to version control.

**In this project**, `profiles.yml` is deliberately kept **inside** the
project folder (not `~/.dbt/`) for reproducibility across machines, but
uses `env_var()` to pull actual secret values from environment
variables rather than hardcoding them:
```yaml
password: "{{ env_var('POSTGRES_PASSWORD') }}"
```
This is a middle ground: the *structure* of the connection config is
version-controlled and reproducible, but the actual secret values never
appear in the committed file itself — set at runtime via
`DBT_PROFILES_DIR` and environment variables instead.

## 11.7 Staging vs. Marts Layering Convention

This is a **widely-adopted dbt community convention**, not a
dbt-enforced rule:
- **Staging** — one model per source table, doing only light renaming/
  casting, no business logic. The "translation layer" between raw
  source structure and everything downstream.
- **Marts** — business-facing models: facts, dimensions, and other
  aggregates analysts and BI tools actually query.

**In this project**, `stg_events.sql` is the entire staging layer (a
thin pass-through over `raw_events`); `dim_users.sql`, `dim_products.sql`,
`dim_time.sql`, and `fact_user_sessions.sql` are the marts layer — each
one reading from `{{ ref('stg_events') }}`, never straight from the
source. Interview-ready reasoning: "if a source column ever gets
renamed, I only need to update one staging model, instead of hunting
through every downstream model that happened to reference the old
name directly."

## 11.8 `dbt_utils` and `date_spine()` (Awareness, Not Used Here)

`dbt_utils` is a widely-used community package providing utilities dbt
core doesn't ship natively — most relevantly here, `date_spine()`,
which generates a **complete, contiguous calendar** (every date/hour in
a range, whether or not any data exists for it).

**In this project**, `dim_time.sql` deliberately does **not** use this —
it's built from *observed* `(event_date, event_hour)` combinations in
the data instead, meaning an hour with zero events simply doesn't
appear as a row at all. This is explicitly documented as sufficient at
this project's scale, with `date_spine()` named as "the standard fix if
a full contiguous calendar (including hours with zero events) is needed
later" — again, a deliberate simplicity choice with a named upgrade
path, not an unknown gap.

---

# TOPIC 12 — PostgreSQL as a Warehouse

## 12.1 Connecting via `psql`, Basic Admin Commands

`psql` is Postgres's interactive command-line client.
```bash
docker exec -it lakehouse-postgres psql -U lakehouse -d warehouse
```
Inside the `psql` prompt, backslash commands are Postgres-specific
meta-commands (not SQL):
```
\d raw_events      -- describe a table's structure (columns, types, keys, indexes)
\dt                 -- list all tables in the current schema
\q                   -- quit
```
**In this project**, `\d raw_events` is used repeatedly across
verification gates specifically to confirm the table was actually
created with `event_id` as a `PRIMARY KEY` — the exact constraint the
upsert logic (`ON CONFLICT (event_id)`) depends on existing.

## 12.2 `docker-entrypoint-initdb.d` — One-Time Initialization Scripts

The official Postgres Docker image has a built-in convention: any
`.sql` (or `.sh`) file placed in `/docker-entrypoint-initdb.d/` inside
the container is automatically executed, **in alphabetical order, but
only the very first time the container initializes a genuinely empty
data directory.**

**In this project**:
```yaml
volumes:
  - ./sql/init_raw_events.sql:/docker-entrypoint-initdb.d/01_init_raw_events.sql:ro
  - ./sql/init_gold_tables.sql:/docker-entrypoint-initdb.d/02_init_gold_tables.sql:ro
```
The `01_`/`02_` prefixes control execution order (alphabetical) — not
strictly necessary here since neither script depends on the other, but
a good habit once init scripts start having dependencies between them.
The `:ro` suffix mounts these files **read-only** inside the container
— defensive, since these are one-time setup scripts, not something the
running database should ever modify.

**The gotcha this project explicitly documents**: because this only
runs against an empty data directory, re-running
`docker compose up -d` after the volume already has data does
**nothing** — you must `docker compose down -v` (destroying the
volume) first to force re-initialization. This is a very common
first-time confusion when working with containerized Postgres.

## 12.3 Designing a Serving Schema for BI/dbt Consumption

A "serving" schema is designed for **read performance and analyst
usability**, not for transactional correctness the way an application's
primary database would be. Key differences from an OLTP schema:
- Denormalized (star schema, Topic 5.7) rather than heavily normalized
  — fewer joins needed to answer a typical business question.
- Optimized for `SELECT`/aggregation-heavy read patterns, not
  high-frequency single-row `INSERT`/`UPDATE` transactions.
- Often intentionally redundant (e.g., `dim_products.avg_observed_price`
  duplicates information technically derivable from `raw_events`, but
  pre-computed for query speed and simplicity).

**In this project**, `raw_events` (populated by the sync jobs) plays
the role of a lightweight "landing" table that dbt then transforms into
the actual serving schema (`fact_user_sessions` + dimensions) — the
distinction between "data that landed" and "data shaped for consumption"
is exactly the ELT boundary discussed in Topic 6.1.

## 12.4 Indexes for Downstream Query Patterns

Covered mechanically in Topic 5.5; worth restating with a
warehouse-specific lens: indexes should be chosen based on **how the
data will actually be filtered, joined, or grouped downstream**, not
added reflexively to every column.

**In this project**:
```sql
CREATE INDEX IF NOT EXISTS idx_raw_events_session_id ON raw_events (session_id);
CREATE INDEX IF NOT EXISTS idx_raw_events_user_id ON raw_events (user_id);
CREATE INDEX IF NOT EXISTS idx_raw_events_event_date ON raw_events (event_date);
```
Each index exists because a specific downstream access pattern needs
it: `session_id` — every dbt mart groups by this to build
`fact_user_sessions`; `user_id` — `dim_users` groups by this; `event_date`
— both dbt's `dim_time` and any date-filtered BI query rely on this.
**Interview framing:** "I added indexes based on the columns my dbt
models actually group and join on, not preemptively on every column —
an unused index still costs write performance and storage, so it should
earn its place based on a real query pattern."

---

## Quick self-check before moving to Topic 13 (BI / Dashboards)

- [ ] Explain what `catchup=False` prevents, and why you'd want it here.
- [ ] Explain why `DockerOperator` launches a separate container rather
      than running code inline in Airflow's own process.
- [ ] Explain `max_active_runs=1` and what specific problem it avoids.
- [ ] Explain the difference between `source()` and `ref()` in dbt, and
      why every mart model in this project uses `ref()` exclusively.
- [ ] Explain the difference between a `view` and `table` materialization,
      and why staging uses one and marts use the other.
- [ ] Explain what a dbt `relationships` test actually checks, in terms
      of a `LEFT JOIN`.
- [ ] Explain why `docker-entrypoint-initdb.d` scripts only run once,
      and what command re-triggers them.
- [ ] Explain why this project's indexes are on `session_id`, `user_id`,
      and `event_date` specifically, not chosen at random.

If you can answer all eight fluently, you're ready for Topic 13.

---
---

## Topics 13–16: BI / Dashboards · Testing · Networking Basics · Production/Cloud Awareness

The final stretch. Topics 13–15 close out the hands-on skills the
project actually required; Topic 16 is deliberately conceptual — it's
what lets you talk credibly about how this local project would look in
a real company, which is very often exactly what an interviewer is
probing for when they ask "how would this change at scale?"

---

# TOPIC 13 — BI / Dashboards (Metabase or Similar)

## 13.1 What a BI Tool's Job Actually Is

A BI (Business Intelligence) tool sits **on top of** a warehouse and
gives non-technical (or SQL-averse) users a way to explore data through
charts, dashboards, and saved questions — without writing SQL by hand
every time, and without needing direct database access.

Under the hood, most BI tools (Metabase included) do one of two things:
1. **Native SQL question** — you write raw SQL yourself; the tool just
   runs it and renders the result.
2. **GUI query builder** — the tool generates SQL for you from clicks
   (pick a table, pick columns, pick filters).

**In this project**, all three dashboard questions
(`sql/dashboards/*.sql`) use option 1 — native SQL — specifically
because the underlying metrics (a funnel, a windowed time series, a
top-N ranking) are precise enough that hand-written SQL is clearer and
more maintainable than trying to express the same logic through a GUI
builder.

## 13.2 Connecting a BI Tool to a Postgres Data Source

This is fundamentally the same connection concept as any other
Postgres client (`psql`, dbt, a Spark JDBC connection) — host, port,
database name, username, password.

**In this project**, the connection details specifically use the
Docker service name as the host:
```
Host: postgres     (not localhost)
Port: 5432
Database name: warehouse
```
This ties directly back to Topic 4.7's networking discussion: Metabase
runs in its own container, on the same Docker network as Postgres — so
it must resolve `postgres` (the service name), not `localhost` (which,
from inside Metabase's own container, means Metabase itself).

## 13.3 Visualization Types — Matching the Chart to the Metric Shape

Choosing the right visualization type isn't cosmetic — the wrong choice
actively obscures what the data is telling you.

- **Funnel** — sequential stages where each stage is a subset of the
  previous one (view → cart → purchase). Visually communicates
  drop-off at each stage far better than a bar chart of the same three
  numbers would.
- **Line chart** — a metric over time, where the *trend* matters (is it
  going up, down, spiking?).
- **Table / bar chart** — ranked, discrete comparisons (top 10
  products) where exact values and ordering matter more than a visual
  trend.

**In this project**:
```sql
-- conversion_funnel.sql → visualized as Funnel
select
    count(*) filter (where view_count > 0) as sessions_with_view,
    count(*) filter (where add_to_cart_count > 0) as sessions_with_add_to_cart,
    count(*) filter (where purchase_count > 0) as sessions_with_purchase
from fact_user_sessions;
```
This single query's three columns map directly onto a funnel's three
stages — Metabase auto-detects this shape and offers the Funnel
visualization specifically because the columns represent a
monotonically-decreasing sequence.

## 13.4 Dashboard Staleness — Understanding When Data Actually Updates

A dashboard is only as fresh as the **last time its underlying data was
written**, not the moment you happen to load the page. Understanding
this chain end-to-end is what separates "I made a dashboard" from "I
understand what makes this dashboard live":

Kafka event produced → Spark ingests to Bronze → Bronze→Silver runs
(every 2 min, via Airflow) → Silver→Gold runs (every 2 min) →
Gold→Postgres sync runs (every 2 min) → Metabase queries Postgres
**live**, on every page load/refresh.

**In this project**, this means a dashboard's numbers can lag the
*actual* live event stream by up to roughly the sum of all those
scheduled intervals — a completely reasonable trade-off for a BI
dashboard (which is rarely used for split-second decisions), but worth
being able to explain precisely, rather than vaguely gesturing at "it
updates automatically."

---

# TOPIC 14 — Testing

## 14.1 `pytest` Basics: Fixtures, `assert`, Parametrization

**Fixtures** are reusable setup/teardown code, shared across multiple
tests, declared with `@pytest.fixture`.

```python
@pytest.fixture(scope="module")
def spark():
    session = SparkSession.builder.master("local[1]").appName("test-medallion").getOrCreate()
    yield session
    session.stop()
```
`scope="module"` means this fixture is created **once** and reused
across every test function in the file (instead of once per test) —
important here specifically because starting a `SparkSession` is
relatively expensive; creating a fresh one for every single test
function would slow the whole suite down considerably.

`yield` (instead of `return`) is what makes a fixture support teardown:
everything before `yield` is setup (runs once, before tests that use
this fixture); everything after `yield` is teardown (runs once, after
all tests using this fixture have finished) — here, `session.stop()`
cleanly shuts Spark down.

**`assert`** is Python's plain built-in assertion — pytest specifically
enhances it to show detailed failure output (what the actual vs.
expected values were) without needing special `assertEqual()`-style
methods the way some other testing frameworks require.

```python
def test_conversion_rate_calculation(spark):
    rows = [_row("view_item") for _ in range(8)] + [_row("purchase") for _ in range(2)]
    df = spark.createDataFrame(rows)
    result = build_conversion_rate(df).collect()[0]
    assert result["view_count"] == 8
    assert result["conversion_rate_pct"] == 25.0
```

## 14.2 Mocking (`unittest.mock.patch`)

**Mocking** replaces a real dependency (a database connection, an
external API call) with a fake stand-in for the duration of a test —
letting you test your *own* logic without needing that real dependency
actually available and working.

**In this project**, `tests/unit/test_sync_silver_to_postgres.py`:
```python
@patch("spark_jobs.sync_silver_to_postgres.execute_values")
@patch("spark_jobs.sync_silver_to_postgres.psycopg2.connect")
def test_write_partition_upserts_rows_in_column_order(mock_connect, mock_execute_values):
    mock_connect.return_value = MagicMock()
    rows = [_fake_row(event_id="e1"), _fake_row(event_id="e2")]
    _write_partition(rows)
    assert mock_execute_values.called
    sent_values = mock_execute_values.call_args.args[2]
    ...
```
Neither decorator requires a real Postgres database — `psycopg2.connect`
is intercepted and replaced with a `MagicMock` (an object that silently
accepts any attribute access or method call and records what happened),
so the test can inspect *what the code would have sent to Postgres*
without a database ever actually running. **This is the key insight
about mocking**: you're not testing that Postgres works (that's
Postgres's own test suite's job) — you're testing that *your code*
constructs the right SQL and values to send it.

Decorators stack **bottom-up**: the closest decorator to the function
(`psycopg2.connect`) corresponds to the *first* mock parameter
(`mock_connect`), and the next one up (`execute_values`) corresponds to
the second (`mock_execute_values`) — a common source of confusion
worth internalizing explicitly.

## 14.3 Testing Spark Logic Locally (`local[1]`)

`SparkSession.builder.master("local[1]")` runs Spark entirely within a
**single local process**, using 1 thread, with no actual cluster
(master/worker containers) involved at all.

**In this project**, every Spark-related test file uses this pattern —
`tests/unit/test_stream_ingest.py`, `test_medallion_transforms.py`,
etc. — because it lets you exercise real Spark DataFrame logic
(`parse_and_validate`, `build_conversion_rate`, `enrich`) with real
Spark semantics, in milliseconds, without needing Docker, Kafka, or the
actual cluster running at all. This is the difference between a *unit*
test (fast, isolated, no external dependencies) and an *integration*
test (slower, exercises real infrastructure) — this project's `pytest`
suite is entirely unit tests; the Day-by-day walkthroughs' manual
verification steps (submitting real jobs, checking real Kafka topics)
are the integration-testing layer, done by hand rather than automated.

## 14.4 What's Unit-Testable vs. What Needs a Manual/End-to-End Check

Not everything *can* be unit tested — and knowing the boundary is
itself a skill worth demonstrating.

**In this project**, `dropDuplicatesWithinWatermark` is explicitly
**streaming-only** — calling it on a static (batch) `SparkSession`
DataFrame raises an error, because its entire logic depends on an
active watermark that only exists in a streaming context. This is why
`bronze_to_silver.py`'s `enrich()` function (the batch-safe half) is
unit tested directly, while the dedup behavior itself is verified only
via the Day 4 end-to-end walkthrough — injecting real duplicate events
and confirming they don't appear twice downstream. **Interview
framing:** "I split the batch-safe transformation logic into its own
function specifically so it could be unit tested in isolation, and I'm
explicit in my test file's docstring about which behaviors are only
verifiable end-to-end, rather than pretending full coverage where it
isn't actually possible."

## 14.5 dbt's Built-in Testing Model vs. Python Testing — Where Each Applies

- **`pytest`** tests **code** — does a function, given certain inputs,
  produce the correct output? (`ip_to_region("8.8.8.8")` returns a
  value in `REGIONS`.)
- **dbt tests** test **data** — does the *actual data currently in the
  warehouse* satisfy a constraint? (Every `user_id` in
  `fact_user_sessions` actually exists in `dim_users`.)

These are complementary, not redundant: a `pytest` suite can pass
100% while the warehouse itself has bad data (e.g., a one-off manual
`INSERT` that violated a business rule); dbt tests catch that, because
they run against real, current data every time you run them — not
against a fixed set of hypothetical inputs.

---

# TOPIC 15 — Networking Basics

## 15.1 Ports: Container-Internal vs. Published/Host Ports

Every service listens on a port *inside* its own container. That port
is only reachable from **outside** the container (i.e., from your host
machine's browser or terminal) if it's explicitly **published** —
mapped to a port on the host.

```yaml
spark-master:
  ports:
    - "8080:8080"    # host_port:container_port
```
The format is always `host:container`. If you wrote `"9999:8080"`
instead, you'd reach Spark's UI via `http://localhost:9999`, even
though Spark itself still thinks it's running on port 8080 internally
— the mapping is purely an external addressing convenience, invisible
to the process inside the container.

## 15.2 DNS Resolution Inside a Docker Network

Covered in Topic 4.4/4.7, worth stating as its own standalone fact
since it's asked about directly in interviews: **Docker's embedded DNS
server resolves a Compose service's name to whichever container is
currently running that service**, automatically, without any manual
configuration. This is why `kafka:29092`, `postgres:5432`, and
`spark-master:7077` all work as hostnames from *other* containers on
the same custom network — they're not real DNS records anywhere on the
internet; they only resolve inside this specific Docker network.

## 15.3 Why `localhost` Means Something Different Inside vs. Outside a Container

This is the single most common practical bug this entire project's
walkthroughs warn about, repeatedly, across multiple phases — worth
committing to memory precisely:

`localhost` (or `127.0.0.1`) always means **"this same network
namespace,"** never "the host machine" and never "a sibling container."
- From your **host terminal**: `localhost:9092` reaches Kafka, *only*
  because port 9092 was explicitly published from the container to the
  host.
- From **inside the Spark container**: `localhost:9092` would mean
  "Spark's own container" — which isn't running Kafka at all, so this
  fails. You must use `kafka:29092` (the internal Docker-network
  hostname and port) instead.

**In this project**, this exact distinction is *why* the Kafka service
defines two separate listeners
(`PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092`) — one for
other containers, one for host-machine tools — rather than expecting a
single address to correctly serve both audiences.

---

# TOPIC 16 — Production/Cloud Awareness

This topic is intentionally conceptual — you don't need hands-on cloud
experience to answer these well, but you do need to be able to *map*
what you built locally onto how it would look in a managed/cloud
environment, since that's a nearly guaranteed interview question for
any project built entirely on local Docker infrastructure.

## 16.1 Mapping This Local Stack to Managed Equivalents

| This project (local, free) | Managed cloud equivalent |
|---|---|
| Kafka (Docker, Zookeeper-based) | Confluent Cloud, AWS MSK, Azure Event Hubs |
| Spark (Docker standalone cluster) | Databricks, AWS EMR, Google Dataproc |
| Delta Lake (local filesystem) | Delta Lake on S3/ADLS/GCS (same format, cloud object storage) |
| Airflow (Docker, SequentialExecutor) | Amazon MWAA, Google Cloud Composer, Astronomer |
| Postgres (Docker) | AWS RDS/Aurora, Google Cloud SQL, Azure Database for PostgreSQL |
| Metabase (Docker) | Same tool, hosted (Metabase Cloud) or a managed alternative (Looker, Tableau) |

**Interview framing:** "Everything in this project is built entirely on
open-source tools running locally in Docker, specifically so it's
reproducible without a cloud account — but the architecture maps
directly onto managed equivalents. If I moved this to AWS, for
instance, Kafka becomes MSK, Spark becomes EMR or Databricks, and Delta
Lake's actual file format doesn't change at all — it would just point
at S3 instead of a local Docker volume."

## 16.2 Cost/Ops Trade-offs: Self-Hosted vs. Managed

- **Self-hosted (this project's approach)** — you own every operational
  concern: patching, scaling, failover, backups, monitoring. Zero
  licensing/hosting cost, but real engineering time cost.
- **Managed services** — the provider handles operations (patching,
  scaling, most failover); you pay for that convenience, typically as a
  usage-based fee. Frees engineering time for higher-value work, at the
  cost of less low-level control and genuine spend.

**In this project's specific context**: Databricks Community/Free
Edition (mentioned back in the original project-proposal discussion) is
a managed Spark offering with a free tier — the trade-off there is fair
-use compute limits, versus this project's self-hosted approach, which
has no imposed limits but requires you to actually manage the Spark
cluster yourself.

## 16.3 What a Schema Registry Would Add

Covered conceptually in Topic 6.11 (data contracts); worth restating in
explicitly cloud/production terms. A **schema registry** (Confluent
Schema Registry, AWS Glue Schema Registry) is a centralized service that:
- Stores the official schema for each Kafka topic (often as Avro or
  Protobuf, not plain JSON).
- Enforces **compatibility rules** on schema changes (e.g., a new field
  must have a default value, so old consumers don't break when reading
  new messages).
- Rejects a producer's write outright if its schema doesn't match what's
  registered, catching schema drift **before** bad data ever reaches a
  topic — rather than this project's approach, where a schema mismatch
  is only caught downstream, at Spark's parsing/quarantine stage.

**In this project**, the explicitly-documented gap (two hand-maintained
copies of the same schema, in `producer/schemas.py` and
`spark_jobs/schema.py`, with no automated sync check) is precisely the
problem a schema registry solves at the infrastructure level instead of
relying on developer discipline.

## 16.4 CI/CD Concepts — Automating Testing/Deployment

**CI (Continuous Integration)** — automatically running your test suite
(and other checks: linting, type-checking) every time code is pushed,
*before* it's allowed to merge — catching regressions immediately
rather than discovering them later, potentially in production.

**CD (Continuous Deployment/Delivery)** — automatically deploying code
that passes CI, either straight to production (Continuous Deployment)
or to a stage awaiting manual approval (Continuous Delivery).

**How this project's testing would plug into CI**: every `pytest` file
built across this project's phases (`test_event_factory.py`,
`test_stream_ingest.py`, `test_medallion_transforms.py`,
`test_sync_silver_to_postgres.py`, `test_sync_gold_to_postgres.py`,
`test_geo.py`, `test_clickstream_pipeline_dag.py`) is exactly what a
GitHub Actions (or similar) workflow would run automatically on every
push — a `pytest` step in a CI pipeline is close to a copy-paste of the
verification-gate commands used manually throughout this project's
walkthroughs. What's genuinely missing for real CI/CD here (worth
naming honestly, the same way `LIMITATIONS.md` names other gaps): no
automated pipeline currently runs these tests on push, no automated
Docker image building/pushing to a registry, and no automated
`docker compose up` + integration test step against real containers —
all reasonable "next steps" to mention if asked how you'd productionize
this project further.

---

## Final self-check — all 16 topics

Before considering this study material complete, you should be able to
explain each of the following out loud, in your own words, without
notes:

- [ ] Why a container isn't "just a lightweight VM"
- [ ] Why `docker compose down -v` was needed after adding Gold's Postgres tables
- [ ] The difference between `ON CONFLICT DO NOTHING` and `DO UPDATE`, and when each is correct
- [ ] What "grain" means for a fact table, and how this project verifies it
- [ ] Why Kafka partition keys matter for downstream ordering
- [ ] Why `dropDuplicatesWithinWatermark` only works in a streaming context
- [ ] Why Silver/Gold jobs switched from continuous streaming to `availableNow` batch runs
- [ ] What Delta Lake's transaction log actually is, and how this project reads it without Spark
- [ ] Why `max_active_runs=1` matters for this specific DAG
- [ ] The difference between `ref()` and `source()` in dbt, and why it matters for the lineage graph
- [ ] Why this project's Postgres indexes are exactly the three they are, not arbitrary
- [ ] Why a BI dashboard's "live" data still lags the actual event stream, and by roughly how much
- [ ] The difference between what pytest tests and what dbt tests test
- [ ] Why `localhost` means something different inside vs. outside a container
- [ ] How this entire local stack maps onto managed cloud equivalents
- [ ] What a schema registry would add over this project's current unenforced contract

This is the complete map — 16 topics, beginner to advanced, every one
tied back to a real decision made somewhere in this project. The
project itself, plus your ability to explain *why* each piece is built
the way it is, is the actual interview preparation — this document is
just the reference to make sure nothing in that explanation is shaky.
