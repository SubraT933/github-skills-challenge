# AIOps Assessment Project

This project simulates a lightweight AIOps workflow for monitoring a service, identifying abnormal behavior from metrics and logs, generating anomaly events, and moving those events through an in-memory event stream.

## Scenario and purpose

The service being monitored is a payment-processing application. The operational data includes timestamps, service identifiers, latency, CPU and memory usage, log levels, and message text. The problem being addressed is that a service can appear healthy in normal intervals yet degrade during brief spikes in latency or resource usage, which may be linked to errors in application logs.

AIOps is used here to connect operational telemetry to a detection pipeline and event-stream workflow so anomalies can be identified early and processed downstream as actionable events.

## Operational data

The repository data file at `data/service_data.json` contains synthetic operational telemetry for a service. Each record includes:

- `timestamp`: when the observation was recorded
- `service`: the name of the service
- `response_time_ms`: a performance metric
- `cpu_percent`: a resource metric
- `memory_percent`: another resource metric
- `log_level`: the severity recorded in the log
- `message`: the corresponding log message

### Metrics vs logs

Metrics are the numeric fields:
- `response_time_ms`
- `cpu_percent`
- `memory_percent`

Log information is represented by:
- `log_level`
- `message`

### Normal versus unusual observations

Normal observations are the early entries with low response times, moderate CPU and memory percentages, and informational log messages such as "Payment request processed successfully".

Unusual observations are the entries with high latency and high utilization, especially the records around the timeout events, where `response_time_ms` rises sharply and `log_level` becomes `ERROR`.

## Anomaly detection findings

The detection logic in `src/anomaly_detector.py` flags an anomaly when any of the following are true:

- response time exceeds the configured threshold
- CPU usage exceeds the threshold
- memory usage exceeds the threshold
- an error log is present

For this dataset, the relevant anomalies are the records around `2026-09-20T10:05:00` and `2026-09-20T10:06:00` for the payment-service. These records were flagged because they show elevated latency, heavy resource usage, and error-level log messages.

The final AIOps pipeline reports two anomalies in the sample dataset.

## Event-processing workflow

The event flow implemented by the repository is:

Operational Data -> Anomaly Detection -> Event -> Producer -> Topic -> Consumer -> AIOps Output

The role of the main components is:

- `AnomalyDetector`: inspects telemetry and decides whether an observation is anomalous
- `EventProducer`: creates and publishes anomaly events
- `EventTopic`: in-memory message topic that stores emitted events
- `EventConsumer`: reads events from the topic and hands them to downstream processing
- `run_pipeline()`: orchestrates the end-to-end workflow

## Issues identified and corrected

The provided workflow had three issues that prevented the intended AIOps flow from working correctly:

1. The detector was checking for `WARNING` logs instead of `ERROR` logs.
   - Affected component: `AnomalyDetector`
   - Fix: check for `ERROR` and record the reason "Error log detected"

2. The producer and consumer were not operating on the same topic.
   - Affected component: `run_pipeline()`
   - Fix: use a shared `EventTopic("anomaly-events")` instance for both producer and consumer

3. The consumer returned messages without draining the topic.
   - Affected component: `EventConsumer`
   - Fix: clear the topic after reading the messages so the processing pipeline behaves consistently

## Validation and execution

These commands were used to validate the project locally:

```bash
py -m pytest -q
py -m pytest --cov=src --cov-report=term-missing -q
py src/aiops_pipeline.py
```

### Result summary

- Tests: 11 passed
- Pipeline result: 10 records processed, 2 anomalies detected, 2 events consumed
- Output confirmed the event flow from anomaly detection to downstream consumption

## Limitation and improvement

One limitation of this AIOps approach is that it is rule-based and uses static thresholds. In real environments, baseline behavior changes over time, so a more robust implementation could use adaptive thresholds, moving averages, or a statistical anomaly model.

## Reproduction steps

1. Clone or open the repository in GitHub Codespaces.
2. Confirm you are working in your fork, not the original repository.
3. Open a terminal in the project root.
4. Install dependencies:
   ```bash
   py -m pip install -r requirements.txt
   ```
5. Run the validation suite:
   ```bash
   py -m pytest -q
   ```
6. Run the end-to-end pipeline:
   ```bash
   py src/aiops_pipeline.py
   ```
7. Review the output to confirm anomaly detection and event flow.

## File overview

- `data/service_data.json`: synthetic operational data
- `src/anomaly_detector.py`: detects anomalies from metrics and logs
- `src/event_topic.py`: in-memory event topic implementation
- `src/event_producer.py`: publishes anomaly events
- `src/event_consumer.py`: consumes anomaly events
- `src/aiops_pipeline.py`: orchestrates the workflow
- `tests/test_aiops_pipeline.py`: validation for anomaly detection and event flow
- `tests/calculations_test.py`: validation for the calculation helpers

## Final status

Tasks 1 through 8 are completed in the repository and validated locally. Step 9 (commit and push to GitHub) is intentionally left out as requested.

