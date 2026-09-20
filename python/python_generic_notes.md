# Python Reference Notes

A consolidated reference of the Python modules, idioms, and patterns commonly used in data engineering work (ETL pipelines, AWS/boto3 automation, database access, logging, testing, and performance). Code snippets are cleaned up from working examples and annotated.

---

## Table of Contents

1. [System & Environment (`os`, `sys`)](#1-system--environment-os-sys)
2. [CLI Arguments (`argparse`)](#2-cli-arguments-argparse)
3. [AWS SDK (`boto3`, `botocore`)](#3-aws-sdk-boto3-botocore)
4. [Date & Time (`datetime`, `time`)](#4-date--time-datetime-time)
5. [File Handling, Context Managers, `pathlib` & JSON](#5-file-handling-context-managers-pathlib--json)
6. [CSV (stdlib `csv`)](#6-csv-stdlib-csv)
7. [Running External Processes (`subprocess`, `os.system`)](#7-running-external-processes)
8. [Logging](#8-logging)
9. [String Handling & f-strings](#9-string-handling--f-strings)
10. [Control Flow / Error Handling & Custom Exceptions](#10-control-flow--error-handling--custom-exceptions)
11. [Collections (`defaultdict`, `Counter`, `namedtuple`)](#11-collections-defaultdict-counter-namedtuple)
12. [AWS Lambda Handler](#12-aws-lambda-handler)
13. [Pandas & Pandas I/O](#13-pandas--pandas-io)
14. [Database Access (`sqlalchemy`, `psycopg2`)](#14-database-access-sqlalchemy-psycopg2)
15. [Functional Tools: `map`, `lambda`, `filter`, `functools.reduce`](#15-functional-tools)
16. [Comprehensions & Generator Expressions](#16-comprehensions--generator-expressions)
17. [Decorators](#17-decorators)
18. [Regular Expressions (`re`)](#18-regular-expressions-re)
19. [Math](#19-math)
20. [Concurrency: `multiprocessing` & `threading`](#20-concurrency-multiprocessing--threading)
21. [HTTP Requests & REST APIs (`requests`)](#21-http-requests--rest-apis-requests)
22. [Type Checking (`mypy`) & `*args/**kwargs`](#22-type-checking-mypy--argskwargs)
23. [Testing (`pytest`)](#23-testing-pytest)
24. [Debugging & Introspection (`traceback`, `id`)](#24-debugging--introspection)
25. [Packaging, `pip`, `venv` & `requirements.txt`](#25-packaging-pip-venv--requirementstxt)
26. [Performance & Profiling (`timeit`, `psutil`, `sum` vs `np.sum`)](#26-performance--profiling)
27. [References](#27-references)

---

## 1. System & Environment (`os`, `sys`)

### `os` — interact with the operating system

The **Python `os` module** serves as an abstraction layer between your Python code and the underlying host operating system. It translates universal Python commands into platform-specific instructions, allowing the exact same script to interact with **Windows file structures** and **Unix (Linux/macOS) file structures** without crashing.

---

### Key Submodules & Environmental Tools

#### 1. `os.path` (The String Architect)
* **Purpose:** Handles the linguistic and structural assembly of file paths. It **does not interact with the system binary `PATH`**.
* **Key Behavior:** It dynamically translates directory separators based on the host platform.
* **Why it matters:** It prevents cross-platform crashes by automatically swapping backslashes (`\`) for Windows and forward slashes (`/`) for Unix.

#### 2. `os.environ` (The System Config Mapping)
* **Purpose:** Reads and modifies the operating system's internal environment variable tables. 
* **Key Behavior:** This is where you access system properties like the active user home directory or the system's execution **`PATH`** variable.
* **Why it matters:** It allows you to look up where the OS searches for executable binaries (`os.environ['PATH']`) or securely fetch hidden application API keys.

#### 3. `os` Top-Level Functions (The Command Executioner)
* **Purpose:** Performs direct file-system actions (navigating, renaming assets) and basic shell execution via legacy tools like `os.system()`.
* **Key Behavior:** Maps Python functions directly to native OS utility commands or host shells. 
* **Why it matters:** Standard actions like `os.listdir()` work cleanly across platforms, but executing shell commands via `os.system()` passes raw strings blindly to the host shell—making it highly platform-dependent and blind to command outputs.

---

####  Top `os` Commands Currently Used

Based on community developer references and tutorials, these are the most heavily utilized `os` commands for day-to-day automation:

* **`os.getcwd()`**: Returns the Current Working Directory. Crucial for debugging exactly where your Python script thinks it is executing.
* **`os.listdir(path)`**: Returns a Python list containing the names of the entries in the directory given by the path.
* **`os.walk(path)`**: A powerful recursive directory crawler. It maps entire file trees by yielding a 3-tuple `(dirpath, dirnames, filenames)` for every directory it scans.
* **`os.makedirs(path, exist_ok=True)`**: Supercharged folder creation. Unlike `os.mkdir()`, this creates nested directories (e.g., `folder/subfolder/file`) and won't throw an error if the directory already exists when `exist_ok=True` is passed.
* **`os.path.exists(path)` / `os.path.isfile(path)`**: Used constantly in conditional statements (`if`) to verify files are physically present before opening them.
* **`os.name`**: Returns the platform indicator ('posix' for Unix/Mac, 'nt' for Windows), frequently used to write conditional logic for multi-OS support.

#### Summary of Platform Behavior

| Python Component | What it handles | Windows Translation | Unix Translation | Modern Alternative |
| :--- | :--- | :--- | :--- | :--- |
| **`os.path.join()`** | Building file paths | Uses `\` separation | Uses `/` separation | `pathlib.Path` |
| **`os.environ['PATH']`** | Finding system binaries | Parses strings split by `;` | Parses strings split by `:` | *None (Standard variable)* |
| **`os.getcwd()`** | Identifying current location | Executes native `cd` tracker | Executes native `pwd` utility | `pathlib.Path.cwd()` |
| **`os.system(cmd)`** | Running shell commands | Passes command to `cmd.exe` | Passes command to `sh`/`bash`/`zsh` | `subprocess.run()` |



```python
import os

# Read env vars (with default fallback) — safer than os.environ['KEY']
region = os.environ.get('AWS_REGION', 'us-east-1')

# Direct access (raises KeyError if missing)
region = os.environ['AWS_REGION']

# Run a shell command (returns exit status). subprocess is preferred.
status = os.system('dir')

# List directory contents
files = os.listdir(os.path.expanduser("~"))   # expands "~" to the home dir

# File metadata
size = os.path.getsize("data.csv")             # bytes
```

**Walking a directory tree** — very common in ETL for discovering files:

```python
import os

for root, dirs, files in os.walk("configuration"):
    for f in files:
        print("joined :", os.path.join(root, f))
        print("relpath:", os.path.relpath(root, '/home/dir'))
```

### `sys` — interpreter & runtime

#### Python `sys` Module Summary

While the `os` module focuses on interacting with the external operating system, the **Python `sys` module** provides deep access to the **Python Interpreter runtime environment itself**. It allows your code to inspect internal interpreter settings, manage memory allocations, handle system execution constraints, and process arguments fed straight from the terminal. 

---

#### ⚠️ The Naming Trap: Breaking Down the 3 "Paths"

Because Python uses the word "Path" in multiple places, it is incredibly easy to confuse them. Here is the definitive breakdown:

* **`os.path` (The Code Tool)**: A code-formatting utility. It is used exclusively to stitch strings together safely into platform-specific folder links (e.g., `folder/subfolder`). It doesn't look things up; it just manipulates text strings.
* **`os.environ['PATH']` (The OS Execution Path)**: An Operating System configuration. This is a list of directories where Windows, Mac, or Linux searches for **executable binary programs** (like `git.exe` or `python.exe`) when you type a command in the terminal.
* **`sys.path` (The Python Module Finder)**: An internal Python runtime list. This is the exact list of folders where the Python interpreter searches for **`.py` scripts or packages** whenever you type an `import` statement in your code.

---

#### Key Components & Environmental Tools

#### 1. Command-Line Arguments (`sys.argv`)
* **Purpose:** Captures arguments passed into your program via the command line interface (CLI).
* **Key Behavior:** It structures all user inputs into a Python list of strings. The very first item, `sys.argv[0]`, is always the name of the script file itself.
* **Why it matters:** It lets you build configurable command-line scripts without hardcoding values inside your text files.

#### 2. Standard Streams (`sys.stdin`, `sys.stdout`, `sys.stderr`)
* **Purpose:** Intercepts standard file pipes for reading input, outputting standard messages, or logging distinct application bugs.
* **Key Behavior:** Maps straight to file-like stream objects that Python interacts with under the hood. 
* **Why it matters:** You can forcefully write to `sys.stderr` to throw high-priority errors to the terminal console, or redirect `sys.stdout` to silently dump output straight into a log file.

#### 3. Interpreter Controls (`sys.exit`, `sys.modules`, `sys.path`)
* **Purpose:** Dictates how modules load, where scripts lookup logic, and when execution terminates.
* **Key Behavior:** Interacts directly with the interpreter memory workspace. For example, modifying `sys.path` dynamically adds new directories for Python to scan during an `import`.
* **Why it matters:** It allows scripts to deliberately stop executing via `sys.exit()` or dynamically manipulate structural dependencies at runtime.

---

#### Top `sys` Properties & Methods Currently Used

* **`sys.argv`**: The universal list tracker for any parameters added during terminal execution.
* **`sys.exit(status)`**: Standard mechanism to terminate a Python program. Passing `0` indicates successful execution, while integers greater than `0` explicitly broadcast failure flags to parent systems.
* **`sys.path`**: A list of string directory tracks where Python actively hunts for modules during execution.
* **`sys.platform`**: Returns the specific build string of the system engine (e.g., `'win32'`, `'linux'`, or `'darwin'` for macOS). Extremely accurate for precise platform sorting.
* **`sys.modules`**: A dictionary cache containing every single module imported since the shell started up.
* **`sys.getsizeof(object)`**: Returns the exact footprint size of an item in bytes, vital for tracing memory leaks or scanning system performance overhead.

---

#### Summary of System Behavior

| Python Component | What it handles | Typical Windows Metric | Typical Unix Metric | Modern Alternative |
| :--- | :--- | :--- | :--- | :--- |
| **`sys.platform`** | Build detection string | Returns `'win32'` | Returns `'linux'` or `'darwin'` | `platform.system()` *(More descriptive)* |
| **`sys.argv`** | Script parameters | Captures arguments passed from CMD/Powershell | Captures arguments passed from Bash/Zsh | `argparse` *(For complex CLI parsing)* |
| **`sys.path`** | Module lookups | Includes `C:\...` tracks | Includes `/usr/local/lib/...` tracks | `PYTHONPATH` variable overrides |


```python
import sys

print(sys.platform)                 # e.g. 'linux', 'win32'
print(sys.version)                  # Python version string

# Add locations to the module search path (PYTHONPATH at runtime)
sys.path.append("/some/path")
sys.path.extend(["/path/a", "/path/b"])
```

**Reading command-line args safely (quick & dirty — prefer `argparse`, section 2):**

```python
import sys

try:
    arg1 = sys.argv[1]
except IndexError:
    raise SystemExit(f"Usage: {sys.argv[0]} <string_to_reverse>")
```

**Combining `os.system` exit status with `sys.exit`:**

```python
import os, sys

cmd = 'dir'
status = os.system(cmd)
if status != 0:
    print(f"FAILED! cmd was '{cmd}'")
    sys.exit(1)
```

---

## 2. CLI Arguments (`argparse`)

The standard way to build command-line ETL scripts — handles parsing, types, defaults, help text, and validation for you.

Handling Python Inputs: Terminal Parsers vs. Function Parameters: When handling inputs in Python, developers often confuse **terminal configurations** (inputs passed from the command line) with **function definitions** (inputs passed inside the Python code itself). 

---

### The Structural Differences

#### Terminal Input Parsers (CLI Tools)
* **`sys.argv` (The Raw List)**: A basic list built into the `sys` module containing the raw text strings typed into the terminal. It provides no automatic parsing, flags, or data type casting.
* **`argparse` (The Feature-Rich Parser)**: A built-in standard library module explicitly designed for building professional Command Line Interfaces (CLIs). It handles user flags (e.g., `-v`, `--verbose`), generates automated help menus (`--help`), and handles data-type validation automatically.

#### Internal Function Parameters (Code Architecture)
* **`*args` (Positional Argument Pack)**: Used inside function definitions to allow the function to accept an **arbitrary number of positional arguments** (passed as an unpacked tuple).
* **`**kwargs` (Keyword Argument Pack)**: Used inside function definitions to allow the function to accept an **arbitrary number of named keyword arguments** (passed as an unpacked dictionary).

To see how these work together, here is a unified script called `process_data.py`. This script uses `argparse` to parse flags from the terminal, peeks at `sys.argv` behind the scenes, and then passes that parsed data straight into a function that utilizes `*args` and `**kwargs`.

```python
import sys
import argparse

# 1. Internal function using *args and **kwargs
def calculate_metrics(*args, **kwargs):
    print("\n--- Inside calculate_metrics() ---")
    print(f"Captured *args (Tuple of arbitrary values): {args}")
    print(f"Captured **kwargs (Dictionary of configurations): {kwargs}")
    
    # Example operation
    total = sum(args)
    print(f"Sum of positional args: {total}")
    if kwargs.get("multiply_by_two"):
        print(f"Configured Result: {total * 2}")

# 2. Parsing inputs using argparse and inspecting sys.argv
def main():
    # Let's peek at sys.argv first to see the raw terminal input
    print(f"Raw sys.argv contents: {sys.argv}")
    
    # Setup argparse for safe, professional CLI input
    parser = argparse.ArgumentParser(description="A sample processing script.")
    
    # Define an argument that expects numbers (integers)
    parser.add_argument('--numbers', nargs='+', type=int, help='A list of numbers to process')
    
    # Define a boolean flag switch
    parser.add_argument('--double', action='store_true', help='Double the total sum')
    
    # Parse the arguments
    parsed_args = parser.parse_args()
    
    if parsed_args.numbers:
        # Pass parsed arguments dynamically into our function
        # *parsed_args.numbers unpacks the list into positional *args
        # multiply_by_two is passed cleanly as a keyword option for **kwargs
        calculate_metrics(*parsed_args.numbers, multiply_by_two=parsed_args.double)
    else:
        print("\nNo numbers provided. Run with --help to see options.")

if __name__ == "__main__":
    main()
```

Second example

```python
import argparse

def parse_args():
    parser = argparse.ArgumentParser(description="Run the ingestion job.")
    parser.add_argument("input", help="Path to the input file")              # positional
    parser.add_argument("-o", "--output", default="out.csv", help="Output path")
    parser.add_argument("--date", required=True, help="Run date YYYY-MM-DD")
    parser.add_argument("--limit", type=int, default=1000, help="Max rows")
    parser.add_argument("--dry-run", action="store_true", help="Don't write output")
    return parser.parse_args()

if __name__ == "__main__":
    args = parse_args()
    print(args.input, args.output, args.date, args.limit, args.dry_run)
```

```bash
python job.py data.csv --date 2024-01-01 --limit 500 --dry-run
python job.py --help        # auto-generated usage/help
```

---

## 3. AWS SDK (`boto3`, `botocore`)

`boto3` is the high-level AWS SDK for Python. `botocore` is the lower-level core it's built on.

### Credentials & region setup

`~/.aws/credentials`:
```ini
[default]
aws_access_key_id = YOUR_KEY
aws_secret_access_key = YOUR_SECRET
```

`~/.aws/config`:
```ini
[default]
region = us-east-1
```

### CloudWatch example

```python
import boto3

cw = boto3.client('cloudwatch')
response = cw.put_metric_data(
    Namespace='MyApp',
    MetricData=[{'MetricName': 'JobsProcessed', 'Value': 42}],
)
```

### `client` vs `resource` vs `session`

- **client** — low-level, 1:1 with the AWS API. Returns dicts.
- **resource** — higher-level, object-oriented abstraction over client.
- **session** — holds config (region, credentials); use it to override defaults.

```python
import boto3

# client: low-level API calls returning dicts
client = boto3.client('s3')
response = client.list_objects_v2(Bucket='mybucket')
for content in response.get('Contents', []):
    obj = client.get_object(Bucket='mybucket', Key=content['Key'])
    print(content['Key'], obj['LastModified'])

# resource: object-oriented
s3 = boto3.resource('s3')
bucket = s3.Bucket('mybucket')
for obj in bucket.objects.all():
    print(obj.key, obj.last_modified)

# session: override region/credentials explicitly
west = boto3.Session(region_name='us-west-2')
east = boto3.Session(region_name='us-east-1')
backup_s3 = west.resource('s3')
video_s3  = east.resource('s3')
```

### `botocore` directly

```python
import botocore.session
from botocore import exceptions
from botocore.client import Config

session = botocore.session.get_session()
client = session.create_client('ec2')
print(client.describe_instances())
```

---

## 4. Date & Time (`datetime`, `time`)

```python
from datetime import datetime
import datetime as dt

# Parse a string into a datetime object
start = datetime.strptime("2020-09-15 12:43:58", '%Y-%m-%d %H:%M:%S')

# "Now" helpers
today  = dt.datetime.now().date()
suffix = dt.datetime.now().strftime("%y-%m-%d-%H-%M")   # good for file names

# Date arithmetic with timedelta
one_day   = dt.timedelta(days=1)
some_date = dt.date(2017, 8, 19)
prior_day = today - one_day
```

```python
import time

time.ctime()          # human-readable current time
time.strptime(...)    # string -> struct_time
time.mktime(...)      # struct_time -> Unix timestamp (float), useful for math
```

> Rule of thumb: use `datetime` for calendar logic and formatting; use `time` when you need Unix timestamps for arithmetic.

---

## 5. File Handling, Context Managers, `pathlib` & JSON

### Context managers (`with`) — the idiomatic way

A context manager guarantees cleanup (closing files, releasing locks, closing DB connections) even if an exception is raised.

```python
# Preferred: file is closed automatically, even on error
with open(path, 'w') as f:
    f.write(report_json)

# Read all lines
with open(path) as f:
    for line in f:            # streams line-by-line, memory-friendly
        process(line)
```

### `pathlib` — modern, object-oriented paths

```python
from pathlib import Path

p = Path.home() / "logs" / "run.txt"
p.parent.mkdir(parents=True, exist_ok=True)   # create dirs
p.write_text("hello")                          # write
text = p.read_text()                           # read
print(p.exists(), p.suffix, p.stem, p.name)
for csv_file in Path("data").glob("*.csv"):     # find files
    print(csv_file)
```

### JSON

```python
import json

# Decode JSON from a file into Python objects (list/dict)
with open("config.json") as f:
    config = json.load(f)
script_arg = config["args"]["script-arguments"]

# String <-> object
obj = json.loads('{"a": 1}')          # str  -> dict
text = json.dumps(obj, indent=2)      # dict -> str

# Write JSON to a file
with open("out.json", "w") as f:
    json.dump(obj, f, indent=2)
```

---

## 6. CSV (stdlib `csv`)

When you can't or shouldn't pull in pandas (small scripts, streaming huge files row-by-row, Lambda cold-start budgets).

```python
import csv

# Read as lists
with open("data.csv", newline="") as f:
    reader = csv.reader(f)
    header = next(reader)
    for row in reader:
        print(row)            # ['a', 'b', 'c']

# Read as dicts keyed by header
with open("data.csv", newline="") as f:
    for record in csv.DictReader(f):
        print(record["column_name"])

# Write
with open("out.csv", "w", newline="") as f:
    writer = csv.DictWriter(f, fieldnames=["id", "name"])
    writer.writeheader()
    writer.writerow({"id": 1, "name": "alice"})
```

---

## 7. Running External Processes

### `subprocess` (preferred)

```python
import subprocess

# Capture command output as bytes
response = subprocess.check_output(job_ids_cmd, shell=True)

# Safer: pass args as a list (avoids shell-injection), capture text
result = subprocess.run(
    ["ls", "-l", "/tmp"],
    capture_output=True, text=True, check=True,
)
print(result.stdout)
```

### `os.system` (simple, returns exit code only)

```python
import os
status = os.system('ls -l')
```

> Prefer `subprocess.run([...], check=True)` with a list of args over `shell=True` string commands — it avoids shell-injection risks and gives structured output.

---

## 8. Logging

Structured logging is essential in pipelines — never rely on `print()` for production ETL.

```python
import logging
import os

# 1. Create logger
logger = logging.getLogger(__name__)
logger.setLevel(logging.DEBUG)

# 2. Formatter
formatter = logging.Formatter('%(asctime)s - %(name)s - %(levelname)s - %(message)s')

# 3. Console handler
ch = logging.StreamHandler()
ch.setLevel(logging.DEBUG)
ch.setFormatter(formatter)

# 4. File handler (create the dir/file first if needed)
filesuffix = "24-01-01-12-00"
logfile = os.path.expanduser(f"~/logs/log{filesuffix}.txt")
os.makedirs(os.path.dirname(logfile), exist_ok=True)
fh = logging.FileHandler(logfile)
fh.setLevel(logging.DEBUG)
fh.setFormatter(formatter)

# 5. Attach handlers
logger.addHandler(ch)
logger.addHandler(fh)

# 6. Use it (lazy % formatting is preferred over f-strings in logging calls)
logger.info('Executing: %s', "get_job_runs_command")
logger.error('Upstream service threw an exception while updating profile definition')
```

> For simple scripts, `logging.basicConfig(level=logging.INFO, format=...)` is a one-line alternative.

---

## 9. String Handling & f-strings

```python
# Format / build strings
uri = 'jobRun/-/{0}/{1}'.format(job_id, date)

# f-strings and the format mini-language
name, n, ratio = "alice", 1234567, 0.8642
f"{name}: {n:,}"      # 'alice: 1,234,567'   (thousands separator)
f"{ratio:.2%}"        # '86.42%'             (percentage, 2 dp)
f"{ratio:.3f}"        # '0.864'              (fixed decimals)
f"{n:>10}"            # right-align width 10

# Split / join
key_words = key.split('/')
joined = ':'.join(key_words)

# Case & whitespace methods
s = " test "
s.upper(); s.lower(); s.capitalize(); s.title(); s.strip()

# Character/ordinal helpers
ord('A')     # 65
chr(65)      # 'A'

# Replace substrings
"banana".replace('a', 'X')   # 'bXnXnX'
```

**Real example — parsing an object-storage key into partition metadata:**

```python
key_words = key.split('/')
partition_words_cnt = len(key_words) - 8 + 4
if partition_words_cnt > 4:
    partition_variant = ':'.join(key_words[4:partition_words_cnt])
bundle_type = key_words[len(key_words) - 2]
version     = key_words[len(key_words) - 3]
snapshot    = key_words[len(key_words) - 4]
table_name  = key_words[2] + "." + key_words[3]
```

---

## 10. Control Flow / Error Handling & Custom Exceptions

### Full try/except/else/finally shape

```python
try:
    result = risky_operation()
except ValueError as ex:          # specific first
    logger.error("bad value: %s", ex)
    raise                          # re-raise, preserving traceback
except Exception as ex:           # broad fallback
    logger.exception("unexpected")  # logs full traceback
else:
    logger.info("succeeded: %s", result)   # runs only if no exception
finally:
    cleanup()                      # always runs
```

### Retry loop

```python
MAX_NUM_ATTEMPTS = 5

for attempt in range(1, MAX_NUM_ATTEMPTS):
    try:
        response = subprocess.check_output(update_profile_command, shell=True)
        break
    except Exception as ex:        # Py3: 'as ex', not 'Exception, ex'
        logger.error('Upstream service exception on attempt %d: %s', attempt, ex)
```

### Custom exceptions

```python
class IngestionError(Exception):
    """Raised when an ingestion step fails."""

def load(record):
    if not record.get("id"):
        raise IngestionError(f"missing id in record: {record}")
```

---

## 11. Collections (`defaultdict`, `Counter`, `namedtuple`)

Workhorses for data munging.

```python
from collections import defaultdict, Counter, namedtuple

# defaultdict — no KeyError; auto-initializes missing keys
groups = defaultdict(list)
for row in rows:
    groups[row["category"]].append(row)      # no need to check existence

# Counter — frequency counts
counts = Counter(["a", "b", "a", "c", "a"])
counts.most_common(2)                          # [('a', 3), ('b', 1)]

# namedtuple — lightweight, readable records
Point = namedtuple("Point", ["x", "y"])
p = Point(1, 2)
print(p.x, p.y)                                # attribute access
```

---

## 12. AWS Lambda Handler

An example Lambda triggered by S3 events, writing metadata to DynamoDB:

```python
import json
import urllib.parse
import boto3

print('Loading function')

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table("my_events_table")

def lambda_handler(event, context):
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = urllib.parse.unquote_plus(event['Records'][0]['s3']['object']['key'])

    sync_date  = event['Records'][0]['eventTime']
    event_name = event['Records'][0]['eventName']

    try:
        table.put_item(Item={
            'table_name': table_name,
            'sync_date': sync_date,
            'key_value': key,
            'event_name': event_name,
        })
    except Exception as e:
        print(e)
        print('Error writing to DynamoDB')
        raise e
```

> In Python 3, use `urllib.parse.unquote_plus` (the Python 2 `urllib.unquote_plus` was removed).

---

## 13. Pandas & Pandas I/O

The workhorse for tabular data manipulation.

```python
import pandas as pd
import numpy as np

# --- Reading / writing many formats ---
df = pd.read_csv("data.csv")
df = pd.read_parquet("data.parquet")           # columnar; fast + compact for data lakes
df = pd.read_json("data.json")
df.to_csv("out.csv", index=False)
df.to_parquet("out.parquet")

# --- Chunked reading for files too big for memory ---
total = 0
for chunk in pd.read_csv("huge.csv", chunksize=100_000):
    total += chunk["amount"].sum()

# --- Core operations ---
pd.isna(df['plan_id'])                          # null check
df['category'].unique()                          # distinct values

# Joins / merges
pd.merge(left_df, right_df, how='left')
pd.merge(combined, forecast_data, how='left',
         left_on=['product_id', 'compute_day'],
         right_on=['product_id', 'forecast_day'])

# Boolean filtering (rows where plan_id is NOT null)
df.loc[np.logical_not(pd.isna(df['plan_id']))]

# GroupBy + apply / aggregate
df.groupby('plan_id', as_index=True).apply(process_group)
agg = df.groupby('supplier_code', as_index=False)['target_value'].sum()

# Aggregations
agg.sum(axis=0)
df['amount'].mean()

# Index & column maintenance
df.reset_index(inplace=True)
df.drop(columns=['category'], inplace=True)

# Weighted average via apply + lambda
temp_group.apply(lambda x: weighted_avg(x, 'lead_time_mean'))
```

---

## 14. Database Access (`sqlalchemy`, `psycopg2`)

Core data-engineering task: connect to a database, run queries safely, and move results in/out of pandas.

```python
from sqlalchemy import create_engine, text
import pandas as pd

# Connection string:  dialect+driver://user:pass@host:port/dbname
engine = create_engine("postgresql+psycopg2://user:pass@host:5432/mydb")

# Read query results straight into a DataFrame
df = pd.read_sql("SELECT id, name FROM users WHERE active = true", engine)

# Write a DataFrame to a table
df.to_sql("users_copy", engine, if_exists="replace", index=False)

# Parameterized query — ALWAYS use params, never string-format SQL (SQL injection)
with engine.connect() as conn:
    result = conn.execute(
        text("SELECT * FROM orders WHERE customer_id = :cid AND amount > :amt"),
        {"cid": 42, "amt": 100.0},
    )
    for row in result:
        print(row)
```

Lower-level with `psycopg2` (parameters passed separately — the driver escapes them):

```python
import psycopg2

with psycopg2.connect(host="host", dbname="mydb", user="user", password="pass") as conn:
    with conn.cursor() as cur:
        cur.execute("SELECT * FROM orders WHERE customer_id = %s", (42,))  # NOT % string-format
        rows = cur.fetchall()
```

> **Security:** never build SQL with f-strings/`%`/`.format()` from user input. Use bound parameters (`:name` in SQLAlchemy, `%s` placeholders in psycopg2) so the driver escapes values safely.

---

## 15. Functional Tools

### `map()`

```python
def square(num):
    return num * num

numbers = [1, 2, 3]
squared = list(map(square, numbers))   # [1, 4, 9]
```

### `lambda` (anonymous functions)

```python
double = lambda x: x * 2
double(5)   # 10
```

### `filter` + `map` combined

```python
import math

def is_positive(num):
    return num > 0

def sanitized_sqrt(numbers):
    cleaned = map(math.sqrt, filter(is_positive, numbers))
    return list(cleaned)

sanitized_sqrt([4, -1, 9])   # [2.0, 3.0]
```

### `functools.reduce`

```python
import functools, operator, os, os.path

files = os.listdir(os.path.expanduser("~"))
total_size = functools.reduce(operator.add, map(os.path.getsize, files))
```

> Use `operator.add` to sum with reduce (`operator.sum` does not exist).

### `enumerate` & `zip` — idiomatic looping

```python
for i, value in enumerate(["a", "b", "c"], start=1):
    print(i, value)            # 1 a / 2 b / 3 c

for name, age in zip(["alice", "bob"], [30, 25]):
    print(name, age)
```

---

## 16. Comprehensions & Generator Expressions

```python
numbers = [1, 2, 3]

# List comprehension — builds the whole list in memory
square_list = [num * num for num in numbers]      # [1, 4, 9]

# Dict & set comprehensions
squares_map = {n: n * n for n in numbers}          # {1: 1, 2: 4, 3: 9}
unique_lengths = {len(w) for w in ["a", "bb", "cc"]}  # {1, 2}

# Generator expression — lazy, memory-efficient for large data
square_gen = (num * num for num in numbers)        # <generator object>
square_list2 = list(num * num for num in numbers)  # materialize when needed
```

> For large datasets, prefer **generator expressions** to avoid loading everything into memory.

---

## 17. Decorators

A decorator wraps a function to add behavior (logging, timing, retries, caching) without changing its body. Common in Airflow, Flask, and utility code.

```python
import functools
import time

def timed(func):
    @functools.wraps(func)                 # preserves name/docstring
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        print(f"{func.__name__} took {time.time() - start:.3f}s")
        return result
    return wrapper

@timed
def extract():
    ...

# Built-in caching decorator (memoization)
@functools.lru_cache(maxsize=None)
def expensive(n):
    return n * n
```

---

## 18. Regular Expressions (`re`)

```python
import re

def remove_punctuation(word):
    return re.sub(r'[!?.:;,"()\-]', "", word)

remove_punctuation("...Python!")   # 'Python'

# Core functions
re.findall(r'\d+', string)     # all non-overlapping matches -> list
re.sub(r'a', 'b', string)      # replace
re.search(r'pat', string)      # first match anywhere -> Match or None
re.match(r'pat', string)       # match at START of string
re.split(r',', string)         # split on pattern

# Compiled patterns (reuse for efficiency)
pattern = re.compile('TP')
pattern.findall('TP Tutorialspoint TP')   # ['TP', 'TP']

# Access match groups
m = re.search(r'(\w+)', string)
m.group(0)     # whole match
```

### Character classes & anchors

| Token | Meaning |
|-------|---------|
| `\w` / `\W` | word char / non-word char |
| `\s` / `\S` | whitespace / non-whitespace |
| `\d` / `\D` | digit / non-digit |
| `\1` | backreference to group 1 |

**Backreference example:**

```python
string = "wish you a happy happy birthday"
result = re.search(r'(\w+)\s\1', string)   # finds repeated word "happy happy"
```

### Lookarounds

| Syntax | Meaning |
|--------|---------|
| `(?=...)` | lookahead positive |
| `(?!...)` | lookahead negative |
| `(?<=...)` | lookbehind positive |
| `(?<!...)` | lookbehind negative |

### Named groups & non-greedy

```python
# Named group
m = re.search(r'(?P<year>\d{4})', "2024")
m.group('year')   # '2024'

# Non-greedy: ? after a quantifier makes it lazy
re.findall(r'<.*?>', "<a><b>")   # ['<a>', '<b>']  (not one big match)
```

---

## 19. Math

```python
import math

math.sqrt(9)     # 3.0
math.ceil(2.1)   # 3
math.floor(2.9)  # 2
math.pi          # 3.14159...
```

(See the `filter` + `map` example in section 15 for `math.sqrt` in a pipeline.)

---

## 20. Concurrency: `multiprocessing` & `threading`

### `multiprocessing.Process` — true parallelism (separate processes, bypasses the GIL)

```python
from multiprocessing import Process

p1 = Process(target=worker.process_batch, args=(1, batches[0]))
p1.start()
p1.join()
```

> Best for **CPU-bound** work (heavy computation over large datasets).

### `threading.Thread` — concurrency within one process

```python
from threading import Thread
from time import sleep

class CookBook(Thread):
    def __init__(self):
        Thread.__init__(self)
        self.message = "Hello Parallel Python CookBook!!\n"

    def run(self):               # override run(); called by .start()
        print("Thread Starting\n")
        for _ in range(10):
            print(self.message)
            sleep(2)
        print("Thread Ended\n")

hello = CookBook()
hello.start()
```

> Best for **I/O-bound** work (network/disk waits). Due to the GIL, threads don't speed up CPU-bound Python.

### Thread synchronization primitives

```python
import threading

threading.current_thread().name

lock = threading.Lock()
lock.acquire()
# ... critical section ...
lock.release()

threading.RLock()        # reentrant lock
threading.Semaphore()    # limit concurrent access to N
threading.Condition()    # wait/notify
threading.Event()        # simple flag-based signaling
```

---

## 21. HTTP Requests & REST APIs (`requests`)

### Basic GET

```python
import requests

x = requests.get('https://example.com/api/data')
print(x.text)          # response body as text
print(x.status_code)   # 200, 404, ...
print(x.json())        # parse JSON body (if applicable)
```

### Full REST API example (GET & POST with headers, params, JSON, error handling)

```python
import requests

BASE_URL = "https://api.example.com/v1"
session = requests.Session()                       # reuse connection + headers
session.headers.update({
    "Authorization": "Bearer <TOKEN>",             # placeholder, not a real token
    "Accept": "application/json",
})

# --- GET with query params ---
def get_orders(customer_id, status="open"):
    resp = session.get(
        f"{BASE_URL}/orders",
        params={"customer_id": customer_id, "status": status},
        timeout=10,                                # ALWAYS set a timeout
    )
    resp.raise_for_status()                        # raise on 4xx/5xx
    return resp.json()

# --- POST with a JSON body ---
def create_order(payload: dict):
    resp = session.post(f"{BASE_URL}/orders", json=payload, timeout=10)
    resp.raise_for_status()
    return resp.json()

# --- Robust call with error handling ---
try:
    data = get_orders(customer_id=42)
    new = create_order({"customer_id": 42, "item": "widget", "qty": 3})
except requests.exceptions.HTTPError as e:
    print("HTTP error:", e.response.status_code, e.response.text)
except requests.exceptions.Timeout:
    print("Request timed out")
except requests.exceptions.RequestException as e:
    print("Request failed:", e)
```

### Pagination (common REST pattern)

```python
def get_all_items():
    items, page = [], 1
    while True:
        resp = session.get(f"{BASE_URL}/items",
                           params={"page": page, "per_page": 100}, timeout=10)
        resp.raise_for_status()
        batch = resp.json()["data"]
        if not batch:                              # empty page -> done
            break
        items.extend(batch)
        page += 1
    return items
```

### Retries with backoff (for flaky endpoints)

```python
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

retry = Retry(total=3, backoff_factor=0.5,
              status_forcelist=[429, 500, 502, 503, 504])
session.mount("https://", HTTPAdapter(max_retries=retry))
```

---

## 22. Type Checking (`mypy`) & `*args`/`**kwargs`

### mypy — static type checker

```bash
python3 -m pip install mypy
mypy program.py        # type-checks and prints errors it finds
```

### `*args` / `**kwargs` with type hints

```python
def stars(*args: int, **kwargs: float) -> None:
    # args   -> Tuple[int, ...]
    # kwargs -> Dict[str, float]
    for arg in args:
        print(arg)
    for key, value in kwargs.items():    # use .items() to unpack
        print(key, value)
```

- `*args` collects extra **positional** arguments into a tuple.
- `**kwargs` collects extra **keyword** arguments into a dict.

---

## 23. Testing (`pytest`)

Conventions:
- Test **file** names start with `test_` (e.g. `test_pipeline.py`).
- Test **function** names start with `test_`.
- Use plain `assert` for validation.

```python
import pytest

def test_that_pipeline_executed():
    assert 1 == 1

# Fixtures provide reusable setup
@pytest.fixture
def sample_df():
    import pandas as pd
    return pd.DataFrame({"id": [1, 2], "amount": [10, 20]})

def test_total(sample_df):
    assert sample_df["amount"].sum() == 30

# Parametrize to run one test over many inputs
@pytest.mark.parametrize("value,expected", [(2, 4), (3, 9)])
def test_square(value, expected):
    assert value * value == expected
```

Run with:
```bash
pytest                    # discover & run all test_*.py
pytest test_pipeline.py   # run a single file
pytest -k total -v        # run tests matching "total", verbose
```

---

## 24. Debugging & Introspection

### Object identity

```python
id(obj)            # unique integer identity
hex(id(obj))       # as hex (like the default repr address)
```

### `traceback` — capture/print exceptions

```python
try:
    import pyarrow
except Exception:
    import traceback
    traceback.print_exc()
    raise Exception("failed to import pyarrow")
```

---

## 25. Packaging, `pip`, `venv` & `requirements.txt`

### Virtual environments — isolate dependencies per project

```bash
python3 -m venv .venv           # create
source .venv/bin/activate       # activate (Linux/macOS)
.venv\Scripts\activate          # activate (Windows)
deactivate                      # leave the env
```

### `pip` & `requirements.txt`

```bash
pip install -r requirements.txt
python3 -m pip install <package>
pip freeze > requirements.txt   # capture exact installed versions
```

Pin versions for reproducible builds:
```
pandas==1.1.5
scikit-learn==0.24.1
numpy==1.19.5
sqlalchemy==1.4.46
requests==2.31.0
```

### `*.egg-info` — build metadata generated for a package

```
<PackageName>.egg-info/
├── PKG-INFO              # name, version, summary, author, etc.
├── entry_points.txt      # console_scripts entry points
├── top_level.txt         # top-level importable packages
└── SOURCES.txt           # every file included in the build
```

`entry_points.txt` maps CLI commands to functions:
```ini
[console_scripts]
run_ingestion = <package_name>.<module_name>:main
```

---

## 26. Performance & Profiling

### `timeit` — micro-benchmark small snippets

```python
import timeit
timeit.timeit('sum(range(100))', number=10000)
```

```bash
python3 -m timeit "sum(range(100))"
```

### `psutil` — process & system resource usage

```python
import psutil
p = psutil.Process(pid=6045)
p.cpu_times()
# pcputimes(user=345.45, system=12.8, children_user=220.03, children_system=17.37)
```

### `sum` vs `np.sum`

- Built-in `sum()` iterates in Python — fine for small lists.
- `numpy.sum()` runs vectorized C code — **much faster** on large numeric arrays.

```python
import numpy as np
np.sum(np.arange(1_000_000))   # far faster than sum(range(1_000_000))
```

> NumPy is implemented in C and is computationally faster for large numeric workloads.

---

## 27. References

- boto3 client vs resource vs session: https://stackoverflow.com/questions/42809096/difference-in-boto3-between-resource-client-and-session
- Video reference: https://www.youtube.com/watch?v=Cb2czfCV4Dg
