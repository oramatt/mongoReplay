# mongoReplay

`mongoReplay.py` is a **MongoDB workload replay tool**.  
It takes JSON/JSONL files containing MongoDB operations (captured from profiling, logs, or exports), replays them against a target MongoDB instance, and produces detailed logs and summary reports.

This tool is useful for:
- Testing and validating workloads on different MongoDB environments.
- Debugging issues with captured operations.
- Simulating workloads in **dry-run mode** without executing.
- Analyzing success/failure rates, collections accessed, and operator breakdowns.

---

## Data Source

The replay tool is designed to take as input the output of the MongoDB profiler when enabled with:

```javascript
db.setProfilingLevel(2)
```

This ensures **all operations** are logged.  
The exported profiling data (in JSONL format) can be generated using the companion script [mongoProfileExport.sh](https://github.com/oramatt/mongotools/blob/main/mongoProfileExport.sh).  

That output file is then fed directly into `mongoReplay.py` as the basis for workload replay.

---

## Features

- Works with **MongoDB profiler data** (from `db.setProfilingLevel(2)`).
- Supports **JSONL** and **JSON array** operation files.
- Handles URIs with authentication, TLS, and `$external` auth sources.
- Dry-run simulation mode (`--dry-run`).
- Duplicate insert handling (`skip` or `upsert`).
- Detailed per-operation logs with optional verbose summary.
- Summarized report:
  - Totals: success, failure, skipped
  - Breakdown by operator type
  - Top 5 most frequent operations
  - Operations by collection
  - Failure summary with detailed failed operations

---

## Installation

Clone the repository and install Python dependencies:

```bash
git clone https://github.com/oramatt/mongoReplay.git
cd mongoReplay
pip install -r requirements.txt
```

**Dependencies:**
- Python 3.13+
- `pymongo`
- `tqdm`

---

## Usage

### Generate Profiler Export

Enable profiling in MongoDB:

```javascript
db.setProfilingLevel(2)
```

Export the profiler collection using [mongoProfileExport.sh](https://github.com/oramatt/mongotools/blob/main/mongoProfileExport.sh):

```bash
./mongoProfileExport.sh 
```

### Replay Workload

Run `mongoReplay.py` against the exported profiler data:

```bash
python mongoReplay.py --file /tmp/demo.json --uri "mongodb://localhost:23456/test?retryWrites=false" --dry-run   --verbose-summary
```

With external auth and TLS in an Oracle Database API for MongoDB:

```bash
python mongoReplay.py --file /tmp/demo.json --uri "mongodb://matt:xxxx@127.0.0.1:27017/matt?authMechanism=PLAIN&authSource=$external&retryWrites=false&loadBalanced=true&tls=true&tlsAllowInvalidCertificates=true"
```

---

## Command Line Options

| Option | Description |
|--------|-------------|
| `--file, -f` | Path to JSON/JSONL file with MongoDB operations |
| `--uri, -u` | MongoDB connection URI (must include database) |
| `--dry-run, -d` | Simulate operations without executing |
| `--max-retries, -retry` | Max retry attempts for MongoDB connection (`-1 = unlimited`) |
| `--verbose-summary, -v` | Include full operation log in the summary report |
| `--duplicate-handling, -dh` | How to handle insert duplicates: `skip` or `upsert` |

---

## Workflow

```mermaid
flowchart TD
    A[Enable Profiling: db.setProfilingLevel(2)] --> B[Export profiler data with mongoProfileExport.sh]
    B --> C[Replay tool consumes JSON/JSONL file]
    C --> D[Connect to MongoDB via MongoClient]
    D --> E{Dry Run?}
    E -->|Yes| F[Simulate Operations Only]
    E -->|No| G[Replay Operations on Target DB]

    G --> H[Handle Supported Ops: find, aggregate, insert, stats...]
    G --> I[Skip Unsupported Ops]

    F --> J[Track Results and Counters]
    H --> J
    I --> J

    J --> K[Generate Summary Report]
    K --> L[Log Success, Failures, Skipped Ops]
    L --> M[End: Write Report File]
```

---

## Sample Input

A small example of profiler data in JSONL format is included in this repository as `sample_profiling_data.jsonl`.  
It contains a few representative operations (query, insert, aggregate). Example snippet:

```json
{"op":"query","ns":"test.users","command":{"find":"users","filter":{"age":{"$gt":30}}},"keysExamined":5,"docsExamined":10,"millis":2,"planSummary":"IXSCAN { age: 1 }","ts":"2025-09-26T12:00:00Z"}
{"op":"insert","ns":"test.users","command":{"insert":"users","documents":[{"_id":1,"name":"Alice","age":25}]},"keysInserted":1,"millis":1,"ts":"2025-09-26T12:01:00Z"}
{"op":"command","ns":"test.$cmd","command":{"aggregate":"users","pipeline":[{"$match":{"age":{"$gte":18}}},{"$group":{"_id":"$status","count":{"$sum":1}}}]},"docsExamined":20,"millis":5,"ts":"2025-09-26T12:02:00Z"}
```

This file can be used as a quick test case to validate that `mongoReplay.py` is working in your environment.

---

## Output

At the end of a run, a report file is generated:

```
replay_summary_YYYY-MM-DD_HH-MM-SS.txt
```

This file includes:
- Input file and connection details (with sanitized URI).
- Totals of operations (success, failed, skipped).
- Operator breakdown.
- Top 5 most frequent operations.
- Operations grouped by collection.
- Failure summary by error type.
- Detailed failed operations (full JSON payloads).

---

## License

This project is licensed under the terms of the [LICENSE](LICENSE) file included in this repository.

---

## FAQ

### Can this tool accidentally destroy my database?

Yes — if you do **not** use the `--dry-run` option, the tool will execute operations against the target MongoDB instance.  
Because it replays operations exactly as they were captured, this may include **inserts, updates, or deletes** that modify live data.

- Always start with `--dry-run` until you are certain of the workload and target environment.  
- Never point this tool at a **production system** unless you fully understand the consequences.  
- The [MIT License](LICENSE) that governs this project explicitly disclaims any warranty or liability. Use at your own risk.

### How do I safely test?

Use `--dry-run` to simulate operations without making changes:

```bash
python mongoReplay.py --file profiling_data.jsonl --uri "mongodb://localhost:23456/test" --dry-run
```

This mode will still produce a full report so you can inspect what **would** have happened.
