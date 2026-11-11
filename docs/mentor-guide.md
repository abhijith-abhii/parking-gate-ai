# Code walkthrough and interview guide

Start in app.py: it reads project.json, creates the Flask application and optionally opens a recorded video. lab/web.py defines the request boundary and serializes inference. lab/inference.py converts pixels into boxes or probabilities; it does not decide what a physical actuator should do. lab/tracking.py links boxes by class and overlap, expires tracks and counts line crossings. lab/runtime.py combines inputs, advances the explicit clock and calls the controller. lab/core.py contains deterministic state transitions without a camera or browser dependency. lab/hal.py supplies clocks, sensors, outputs and SQLite persistence. static/app.js displays state and sends explicit user actions.

Read one scenario JSON next. Each object is a command, such as setting sensor values, checking, or advancing time. Run the matching test in tests/test_core.py and change one input. Then inspect evidence/results.json and the raw trace to connect a rule to a measured outcome. The projects share a controller library and regression suite; the repeated 56-test result is the same suite run in each standalone package, not 168 unique tests.

## Decisions and alternatives

Higher resolution increased vehicle F1 from 0.386 to 0.581 at roughly 3.6 times detector latency. Conservative fusion eliminated 15 false-free slot-seconds on the scripted trace, but only 83.33% of slot-seconds received a definite decision. Accuracy on decided slots was 100%; counting unknowns as errors gives 83.33%, below the baseline 93.75%. Reserving uncertainty costs availability.

Python and ONNX make desktop replay practical. A bare-metal RTOS or emulator would not validate camera models without a target-specific peripheral model. A class-aware IoU tracker is simple to inspect but can switch IDs during occlusion; an appearance tracker could improve association at added cost and still would not establish identity or liveness. SQLite keeps events and state atomic, while physical side effects require a separate acknowledgement design.

## Ten interview questions

### 1 Why reserve conflicting spaces?

A false-free decision can admit a vehicle into an unavailable space. Reserving conflict reduces this error at the cost of 16.67% undecided slot-seconds in the synthetic trace.

### 2 Can manual close defeat the interlock?

No. Manual requests still require fresh clear evidence for two seconds; a blocked or invalid observation holds the gate.

### 3 Does HOLD mean the gate reopens?

No. It is a virtual stop state. Motor braking, reversing distance and independent physical interlocks were not validated.

### 4 Why is vehicle F1 different from occupancy accuracy?

Object boxes are matched on COCO photos, while availability is scored on virtual space readings. Neither establishes performance on a calibrated parking camera.

### 5 How do you prevent test leakage?

Freeze selection using validation before reading final test scores. Exact duplicates are grouped for TrashNet; COCO images are disjoint. Scene identifiers are missing, so independence is limited. Future video splits must hold out entire recordings and locations.

### 6 How do you distinguish confidence from accuracy?

Confidence is one model score. Accuracy is a frequency measured against held-out labels. The dashboard displays scores while the evidence files report measured performance.

### 7 Why use an explicit simulation clock?

It makes long timeouts deterministic and replayable. A 600-second step integrates one-second controller ticks. This does not emulate Pi scheduling, camera timing or electrical behavior.

### 8 What happens when saving fails?

SQLite commits an event ID and state together. Failed commits restore controller and inference/tracker snapshots. A real actuator cannot be rolled back this way, so physical integration needs an outbox and acknowledgement protocol.

### 9 How is a retry different from a new event?

Reusing an event_id is idempotent. A new event ID is a new command. Waste item IDs prevent repeat counts within a bounded history; a production system needs a durable item ledger across unlimited retention.

### 10 What would you measure on a Raspberry Pi?

End-to-end latency including capture, missed deadlines under CPU load, thermal throttling, process memory, sensor stale time, restart behavior, supply voltage and motor/LED timing. Host measurements cannot be relabeled as Pi results.

## Modifications to implement yourself

1. Replace example vertical lanes with named calibrated polygons and test a vehicle straddling two spaces.
2. Add a bounded event queue so control ticks can continue while inference is busy; measure worst-case interlock delay.
3. Add independent limit-switch acknowledgement to the virtual actuator and test contradictory limits and restart during motion.

## Evidence backed resume bullets

- Implemented a Raspberry Pi–targeted parking simulator with camera/proximity fusion, explicit gate states and an obstruction interlock that also applies to manual requests.
- Improved vehicle F1 from 0.386 to 0.581 on 78 held-out COCO images by increasing inference resolution, with class-level results and host timing evidence.
- Eliminated 15 false-free slot-seconds and 14 unsafe closure requests in separate synthetic comparisons, reporting 83.33% availability-decision coverage.

Use these after you can explain and reproduce the implementation. Keep simulation and AI assistance clear in the project discussion.
