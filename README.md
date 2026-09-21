# GitHub Challenge

<img src="https://octodex.github.com/images/Professortocat_v2.png" align="right" height="200px" />

Hey there!

Your challenge is ready.
Follow the instructions provided for this challenge and complete the required tasks in this repository.

Make sure your work is committed and pushed to your repository before submission.

Good luck!


---

&copy; 2025 GitHub &bull; [Code of Conduct](https://www.contributor-covenant.org/version/2/1/code_of_conduct/code_of_conduct.md) &bull; [MIT License](https://gh.io/mit)

## AIOps scenario

This repository models a lightweight AIOps workflow for the payment-service application. The goal is to monitor service telemetry and log events, detect abnormal behavior, convert the issue into an event, and confirm that the event travels through a simple event-processing pipeline before reaching downstream handling.

## Operational data

The operational data is stored in [data/service_data.json](data/service_data.json). Each record represents one observation for the payment-service and includes:

- timestamp
- service
- response_time_ms
- cpu_percent
- memory_percent
- log_level
- message

These values capture both the health of the service and the context of the system event at a specific point in time.

## Log and metric observations

### Metrics

The metric fields are:

- response_time_ms
- cpu_percent
- memory_percent

These are numeric measurements of service performance and resource load.

### Log information

The log fields are:

- log_level
- message

The dataset alternates between INFO-level success logs and ERROR-level timeout logs, which is a strong signal that the service is healthy most of the time but experiences a measurable outage or degradation event.

### Normal behaviour

Normal observations include the records from 10:00:00 through 10:04:00 and 10:07:00 through 10:09:00. In those observations:

- response_time_ms is roughly 120-150 ms
- cpu_percent is roughly 42-57%
- memory_percent is roughly 51-57%
- log_level is INFO
- message is Payment request processed successfully

This pattern reflects a healthy, stable service.

### Unusual behaviour

The unusual observations are the records at 10:05:00 and 10:06:00. In those observations:

- response_time_ms rises to 610 and 640 ms
- cpu_percent rises to 75 and 94%
- memory_percent rises to 70 and 91%
- log_level becomes ERROR
- message becomes Payment service timeout / Database connection timeout

This pattern clearly indicates a service degradation event.

## Anomaly-detection findings

Running the provided detector on the operational data produced two anomaly events:

1. 2026-09-20T10:05:00
   - service: payment-service
   - type: ANOMALY
   - reasons: High response time
   - source metrics: response_time_ms = 610, cpu_percent = 75, memory_percent = 70
   - source log: ERROR, Payment service timeout

2. 2026-09-20T10:06:00
   - service: payment-service
   - type: ANOMALY
   - reasons: High response time, High CPU utilization, High memory utilization
   - source metrics: response_time_ms = 640, cpu_percent = 94, memory_percent = 91
   - source log: ERROR, Database connection timeout

The normal records remained below threshold and were not flagged.

## Event-processing flow

The repository uses a lightweight in-memory event-streaming architecture:

Operational Data -> Anomaly Detection -> Event -> Producer -> Topic -> Consumer -> AIOps

The event payload contains enough information for downstream processing to understand why it was flagged. The producer writes the event to the topic, and the consumer reads it back from the topic for AIOps processing.

## Final workflow execution result

I executed the full workflow on the repository dataset and confirmed the complete end-to-end path.

Final execution output:

- records_processed = 10
- anomalies_detected = 2
- events_consumed = 2

Detected anomaly events:

- 2026-09-20T10:05:00 — ANOMALY (High response time)
- 2026-09-20T10:06:00 — ANOMALY (High response time, High CPU utilization, High memory utilization)

This confirms that operational data is processed, abnormal behaviour is detected, the anomaly event is generated, published, consumed, and processed successfully.

## Issues identified and corrected

During the investigation, I identified and corrected the following issues:

1. Package import issue
   - Affected component: [src/__init__.py](src/__init__.py)
   - Cause: Python was not recognizing the project as a package in the test environment.
   - Fix: added the package initializer so imports like `from src.anomaly_detector import AnomalyDetector` work correctly.

2. Incorrect log condition in detection
   - Affected component: [src/anomaly_detector.py](src/anomaly_detector.py)
   - Cause: the detector checked for `WARNING` instead of `ERROR`, so real outages were not being associated with the correct log severity.
   - Fix: corrected the logic so error-level events are properly included in the anomaly reasons.

3. Pipeline path and event-flow issue
   - Affected component: [src/aiops_pipeline.py](src/aiops_pipeline.py)
   - Cause: the workflow was not resolving the data path and event flow consistently from the repository root.
   - Fix: resolved the data path relative to the repo and kept the pipeline on the existing producer/topic/consumer architecture.

## Limitation and possible improvement

A limitation of the current implementation is that it uses fixed thresholds for response time, CPU, and memory. This works for the synthetic dataset but may not adapt to changing baselines in a live system. A possible improvement would be to use adaptive thresholds or a rolling baseline derived from recent service behavior so the detector can respond to gradual drift while still catching genuine incidents.

## Reproduction steps

To reproduce the work in another environment:

1. Open a terminal in the repository root.
2. Install dependencies:
   - `pip install -r requirements.txt`
3. Run the test suite:
   - `python -m pytest -q`
4. Run the end-to-end workflow:
   - `python - <<'PY'
     from src.aiops_pipeline import run_pipeline
     result = run_pipeline('data/service_data.json')
     print(result)
     PY`
5. Confirm the output shows 10 processed records and 2 detected anomalies.

Expected result: the workflow identifies the timeout-related anomalies and emits the corresponding events for downstream AIOps consumption.
