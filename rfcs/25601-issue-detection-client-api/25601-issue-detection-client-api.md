# RFC: Typed Client API to Submit Issue Detection for Traces

| start_date | 2026-09-18 |
| :--- | :--- |
| **mlflow_issue** | https://github.com/mlflow/mlflow/issues/25601 |
| **rfc_pr** | |
| **Author(s)** | [Rohitkanithi](https://github.com/Rohitkanithi), [Adam Gurary](https://github.com/adamgurary), [Joshua Wong](https://github.com/joshuawong-db), [Haoji Tang](https://github.com/tanghaoji) |
| **Date Last Modified** | 2026-09-18 |

---

## Summary

This RFC proposes adding a formal, typed public client API and standard Protobuf RPC endpoint to MLflow for submitting GenAI **Issue Detection** jobs against logged traces.

Specifically, this proposal:
1. Promotes the internal, UI-specific AJAX route (`/ajax-api/3.0/mlflow/issues/invoke`) into an official, versioned Protobuf RPC (`POST /api/2.0/mlflow/issues/invoke`).
2. Introduces `client.submit_issue_detection(...)` on `MlflowClient`, returning a typed `IssueDetectionJob` handle (`job_id`, `run_id`).
3. Exposes `client.search_issues(...)` on `MlflowClient`, enabling callers to programmatically query and filter detected issues by `experiment_id`, `source_run_id`, severity, or category.
4. Preserves server-side credential resolution—ensuring no raw provider API keys or bearer tokens are ever accepted from or transmitted across client networks.

---

## Basic Example

### 1. Submitting an Asynchronous Issue Detection Job
```python
from mlflow import MlflowClient

client = MlflowClient()

# Submit an issue detection job on traces for an experiment
job = client.submit_issue_detection(
    experiment_id="101",
    trace_ids=["tr-7a8b9c", "tr-1d2e3f"],
    categories=["hallucination", "tool_error", "safety"],
    provider="openai",
    model="gpt-4o",
)

print(f"Submitted background job: {job.job_id}")
print(f"Tracking run ID: {job.run_id}")
```

### 2. Searching and Retrieving Detected Issues
```python
# Query issues identified during the detection run
issues = client.search_issues(
    experiment_id="101",
    source_run_id=job.run_id,
    filter_string="severity = 'high'",
)

for issue in issues:
    print(f"Issue: {issue.name} [{issue.severity}]")
    print(f"Description: {issue.description}")
    print(f"Categories: {issue.categories}")
    print(f"Root Causes: {issue.root_causes}")
```

---

## Motivation

### The Problem
MLflow Tracing records detailed spans, inputs, and outputs for GenAI applications and agents. To help teams diagnose quality and operational regressions across production traces, MLflow introduced **Issue Detection** (clustering and identifying recurring failures such as schema deviations, hallucinated tool calls, and model refusals).

However, currently:
- **UI-Coupled Endpoint**: The endpoint that triggers issue detection is exposed only as an ad-hoc, internal AJAX route (`/ajax-api/3.0/mlflow/issues/invoke`) designed strictly for web browser interactions.
- **No Client API**: `MlflowClient` exposes no public methods to trigger issue detection or query persisted issues.
- **Automation Blocked**: Automated evaluation pipelines, continuous integration (CI/CD) testing gates, scheduled batch inspection jobs, and autonomous agents cannot run issue detection programmatically.

### Use Cases
1. **Continuous Integration & Evaluation Gates**:
   An automated test suite logs benchmark agent runs to MLflow Tracing and immediately invokes `client.submit_issue_detection(...)` to verify trace quality before promoting a prompt or agent version to production.
2. **Scheduled Batch Scanning**:
   A scheduled job (e.g., in Airflow, Prefect, or Databricks Workflows) runs periodically to sample newly logged production traces, triggers issue detection, and searches for any `high` or `medium` severity issues.
3. **Autonomous Agent Reflection**:
   Multi-agent systems or self-improving agent frameworks programmatically trigger trace scans on their own session traces to discover tool call failures and adjust prompts or tool schemas dynamically.

### Out of Scope
- Implementing new detector algorithms or modifying clustering logic (this proposal focuses strictly on standardizing the RPC protocol and public client surface around the existing execution infrastructure).
- Modifying UI components (e.g., the trace detail view or review app layouts).

---

## Detailed Design

### Architecture & Component Flow

```
+-------------------------------------------------------------+
|                      Python Client                          |
|  client.submit_issue_detection(experiment_id, trace_ids...) |
+-------------------------------------------------------------+
                              |
                              | Protobuf RPC (POST /api/2.0/mlflow/issues/invoke)
                              v
+-------------------------------------------------------------+
|                 MLflow Tracking Server                      |
|                                                             |
|  1. Authenticate caller & validate experiment permissions:  |
|     validate_can_update_experiment(experiment_id)           |
|                                                             |
|  2. Resolve LLM provider credentials server-side            |
|     (AI Gateway, Secret Store, or Server Environment)       |
|                                                             |
|  3. Launch asynchronous job & create tracking run:          |
|     invoke_issue_detection_job(...) -> job_id, run_id       |
+-------------------------------------------------------------+
                              |
                              v
+-------------------------------------------------------------+
|                     Tracking Store                          |
|  Persists run metadata, tags, and detected Issue entities   |
+-------------------------------------------------------------+
```

---

### 1. Protobuf Protocol Specification

#### Message Definition (`mlflow/protos/issues.proto`)
Add the `SubmitIssueDetection` request and response message:

```protobuf
message SubmitIssueDetection {
  // The ID of the experiment associated with the traces.
  optional string experiment_id = 1 [(validate_required) = true];

  // The list of trace IDs to analyze for issues.
  repeated string trace_ids = 2;

  // The categories of issues to evaluate (e.g. hallucination, safety).
  repeated string categories = 3;

  // Optional provider name (e.g. openai, anthropic, bedrock).
  optional string provider = 4;

  // Optional model name (e.g. gpt-4o, claude-3-7-sonnet).
  optional string model = 5;

  // Optional secret ID for credentials stored on the server.
  optional string secret_id = 6;

  // Optional AI Gateway endpoint name.
  optional string endpoint_name = 7;

  message Response {
    // Unique identifier for the asynchronous background job.
    optional string job_id = 1;

    // MLflow run ID created to track this issue detection execution.
    optional string run_id = 2;
  }
}
```

#### Service RPC Registration (`mlflow/protos/service.proto`)
Register the RPC under `MlflowService`:

```protobuf
rpc submitIssueDetection (mlflow.issues.SubmitIssueDetection) 
    returns (mlflow.issues.SubmitIssueDetection.Response) {
  option (rpc) = {
    endpoints: [
      {
        method: "POST"
        path: "/mlflow/issues/invoke"
        since: {
          major: 3
          minor: 0
        }
      }
    ]
    visibility: PUBLIC_UNDOCUMENTED
    rpc_doc_title: "Submit issue detection for traces"
  };
}
```

---

### 2. Server-Side Execution & Security Model

In `mlflow/server/handlers.py`:
- Map the RPC handler in the dispatch table:
  ```python
  HANDLERS[SubmitIssueDetection] = _invoke_issue_detection_handler
  ```
- **Permission Validation**:
  Submitting an issue detection job creates a tracking run in the target experiment. The handler strictly requires write permissions on the target experiment:
  ```python
  validate_can_update_experiment(experiment_id)
  ```
- **Server-Side Credential Resolution**:
  The client payload accepts provider names, model identifiers, or gateway endpoint names, but **never accepts raw API keys or tokens**. The server resolves credentials via:
  1. Configured MLflow AI Gateway endpoints.
  2. Server environment variables (e.g. `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`).
  3. Server-managed secrets (`secret_id`).
- Returns `SubmitIssueDetection.Response(job_id=job.job_id, run_id=run_id)` wrapped via standard `_wrap_response(...)`.

---

### 3. Store Layer Contracts

In `mlflow/store/tracking/abstract_store.py`:
```python
@abstractmethod
def submit_issue_detection(
    self,
    experiment_id: str,
    trace_ids: list[str],
    categories: list[str],
    *,
    provider: str | None = None,
    model: str | None = None,
    secret_id: str | None = None,
    endpoint_name: str | None = None,
) -> IssueDetectionJob:
    """Submit an asynchronous issue detection job."""
    pass
```

Implementations:
- **`RestStore` & `DatabricksRestStore`**:
  Serialize the request using `message_to_json(SubmitIssueDetection(...))` and dispatch via `self._call_endpoint(...)`.
- **`SqlAlchemyStore`**:
  Because asynchronous job execution requires the server background runner, direct local store calls raise an actionable error:
  ```python
  raise MlflowException(
      "Submitting issue detection is only supported against a remote MLflow tracking server "
      "or when running the MLflow server."
  )
  ```

---

### 4. Python Client API (`MlflowClient`)

Two public methods are added to `mlflow.tracking.MlflowClient`:

#### `client.submit_issue_detection(...)`
```python
def submit_issue_detection(
    self,
    experiment_id: str,
    trace_ids: list[str],
    categories: list[str],
    *,
    provider: str | None = None,
    model: str | None = None,
    secret_id: str | None = None,
    endpoint_name: str | None = None,
) -> IssueDetectionJob:
    """
    Submit an issue detection job on traces asynchronously.

    Args:
        experiment_id: The ID of the experiment containing the traces.
        trace_ids: List of trace IDs to analyze.
        categories: Categories of issues to evaluate (e.g. 'hallucination').
        provider: Optional provider name ('openai', 'anthropic', etc.).
        model: Optional model name ('gpt-4o', etc.).
        secret_id: Optional server secret ID for authentication.
        endpoint_name: Optional MLflow Gateway endpoint name.

    Returns:
        An IssueDetectionJob handle containing job_id and run_id.
    """
```

#### `client.search_issues(...)`
```python
def search_issues(
    self,
    experiment_id: str | None = None,
    filter_string: str | None = None,
    max_results: int | None = None,
    page_token: str | None = None,
    source_run_id: str | None = None,
    include_trace_count: bool = False,
) -> PagedList[Issue]:
    """
    Search for issues matching the given filters.

    Args:
        experiment_id: Optional experiment ID to scope the search.
        filter_string: Optional SQL-like filter string (e.g. "severity = 'high'").
        max_results: Maximum number of issues to return.
        page_token: Token for pagination.
        source_run_id: Optional run ID that discovered the issues.
        include_trace_count: Whether to compute and include affected trace counts.

    Returns:
        PagedList of Issue objects.
    """
```

---

### 5. Entity Model

Exposed in `mlflow.entities`:
- **`Issue`**: Contains `issue_id`, `experiment_id`, `name`, `description`, `status` (`IssueStatus`), `severity` (`IssueSeverity`), `root_causes`, `source_run_id`, `categories`, and `trace_count`.
- **`IssueDetectionJob`**: Lightweight handle containing `job_id: str` and `run_id: str`.
- **`IssueStatus`**: Enum (`pending`, `rejected`, `resolved`).
- **`IssueSeverity`**: Comparable Enum (`not_an_issue`, `low`, `medium`, `high`) supporting `<` and `>` comparisons.

---

## Drawbacks

- Submitting an issue detection job requires an MLflow tracking server with background job execution capability. When running against an embedded local filesystem / SQLite store without a server daemon, `submit_issue_detection` cannot run asynchronously and informs the user to run `mlflow server`.

---

## Alternatives Considered

1. **Keep Untyped AJAX Route with Client Raw HTTP Call**:
   - Having `MlflowClient` make an untyped POST request directly to the internal AJAX endpoint.
   - *Rejected*: Inconsistent with MLflow's architecture. MLflow's client-server communication relies on typed Protocol Buffer RPCs for schema validation, cross-language SDK support, and backward compatibility.
2. **Client-Side Execution**:
   - Having the Python client fetch traces and run the detection logic locally.
   - *Rejected*: Running detectors requires downloading large volumes of raw trace data and distributing LLM provider API credentials to all client environments, creating significant security and bandwidth concerns.

---

## Adoption Strategy

- **Backward Compatibility**: Fully backward compatible. This change is strictly additive. Existing UI flows and tracing endpoints continue to function without interruption.
- **Documentation**: Provide code recipes in the GenAI Tracing & Evaluation documentation demonstrating automated issue detection in CI pipelines and scheduled evaluation workflows.

---

## Open Questions

1. **Blocking vs. Asynchronous API**:
   Should `submit_issue_detection` optionally support a blocking flag (`wait: bool = False`) that polls the job status until completion?  
   *Recommendation*: Keep the initial version asynchronous-only (`submit_issue_detection`) to avoid long-lived HTTP connection drops on large trace batches. A helper like `client.wait_for_issue_detection(job_id)` can be added in a fast follow-up if desired.
