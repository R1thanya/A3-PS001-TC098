# ShieldX Lab (PS-001)
### Android Ransomware-Like Activity Early Warning in a Controlled Lab
**Category:** Cyber Security  
**Team Name:** ShieldX Lab (aiori-hackathon)  
**Deliverable PDF:** Attached

**Youtube Video Link:** (https://youtu.be/gb99IMN3c4A?si=OHHuiOlcNNTKR8sx)

**Pseudocode Snippet:** [`docs/PSEUDOCODE_SNIPPET.md`](docs/PSEUDOCODE_SNIPPET.md)  

---

## 1. Executive Summary & Challenge
Signature-based antivirus tools cannot detect zero-day ransomware because unknown malicious binaries have not yet been cataloged in signature feeds. On Android, application sandboxing prevents security software from directly inspecting the memory of other processes.

However, crypto-ransomware produces an unmistakable file-system behavioural footprint:
- High-frequency write bursts across user files.
- Sharp positive jumps in Shannon entropy ($\Delta H \ge 2.0$) or high absolute entropy ($H \ge 7.20$ bits/byte) resulting from cryptographic encryption (AES / ChaCha20 / XOR).
- Rapid file renaming (`.locked`, `.crypto`) or sequential wipe-and-drop file replacements.

**ShieldX Lab** is a behaviour-based early-warning detector built for an instrumented Android lab testbed. By continuously monitoring file-system telemetry within a designated directory, ShieldX Lab flags anomalous rewrite bursts and terminates **only the controlled test workload** without disrupting device stability or claiming unrealistic OS-wide prevention.

---

## 2. Key Requirements & Boundary Compliance

| Requirement | Implementation in ShieldX Lab | Verification Status |
| :--- | :--- | :--- |
| **Android Emulator / Lab Device** | Google APIs x86_64 AVD (API 34) with documented telemetry. | Configured & Tested |
| **Monitored Directory** | Scoped explicitly to `/sdcard/Download/RansomLab` or app storage. | Strict boundary adherence |
| **2 Benign Workloads** | **Benign 1**: Document autosave / note editor<br>**Benign 2**: Batch camera media download | Verified (0 false positives) |
| **3 Harmless Ransomware Simulations** | **Sim 1**: Rapid in-place overwrite (high entropy)<br>**Sim 2**: Encrypt & rename (`.locked`)<br>**Sim 3**: Wipe & drop encrypted replacement (`.enc`) | Verified (100% detection) |
| **Quantified Evaluation Metrics** | Detection delay (ms), Files modified before alert, Precision, Recall, F1-Score. | Fully quantified |
| **Controlled Workload Termination** | Test workload isolated in `:rogue_workload` secondary process; stopped via targeted PID termination. Main monitor remains active. | Zero collateral impact |
| **No Device-Wide Claims** | Scoped strictly to lab directory; no claims of OS-wide prevention on stock Android. | Boundary respected |

---

## 3. Architecture & Telemetry Pipeline

```
[Android File Storage Event]
            │
            ▼
[Linux inotify / FileObserver]
   - Event Types: CREATE, MODIFY, CLOSE_WRITE, RENAME, DELETE
   - Paths & Timestamp (epoch ms)
            │
            ▼
[Shannon Entropy Calculator H(X)]
   - Chunk-sampled Head, Mid, Tail calculation
   - Entropy Delta: ΔH = H_post - H_pre
            │
            ▼
[Sliding-Window Behavioral Engine] (3000 ms Window)
   - Evaluates write velocity, rename rates, and entropy shifts
   - Dampens benign autosave and linear media sync
            │
            ▼
[Anomaly Score >= 75 (Critical Threshold)]
   - Calculates Detection Delay: t_alert - t_first_suspicious
   - Records Files Modified Before Alert
            │
            ▼
[Controlled Mitigation Manager]
   - Broadcasts immediate cancellation signal
   - Executes Process.killProcess(roguePid) on :rogue_workload
   - Fires high-priority system alert notification
```

---

## 4. Experimental Evaluation & Benchmark Results

The evaluation suite was executed inside the Android API 34 lab environment across a standardized 25-file test corpus:

| Workload Name | Classification | Result | Detection Delay | Files Changed | Files Preserved |
| :--- | :--- | :--- | :---: | :---: | :---: |
| **Benign 1: Doc Autosave / Editor** | Benign | **PASSED (TN)** | N/A | 0 / 25 | 25 (100%) |
| **Benign 2: Batch Media Sync** | Benign | **PASSED (TN)** | N/A | 0 / 25 | 25 (100%) |
| **Sim 1: Rapid In-Place Overwrite** | Malicious | **ALERT (TP)** | **95 ms** | 3 / 25 | **22 (88%)** |
| **Sim 2: Encrypt & Rename (.locked)** | Malicious | **ALERT (TP)** | **80 ms** | 2 / 25 | **23 (92%)** |
| **Sim 3: Wipe & Drop (.enc)** | Malicious | **ALERT (TP)** | **110 ms** | 3 / 25 | **22 (88%)** |

### Mathematical Summary Metrics:
- **Precision:** $\frac{TP}{TP + FP} = \frac{3}{3 + 0} =$ **100.0%**
- **Recall:** $\frac{TP}{TP + FN} = \frac{3}{3 + 0} =$ **100.0%**
- **F1-Score:** **1.00**
- **Mean Detection Delay:** **95.0 ms**
- **Average Files Lost Before Alert:** **2.67 files** (10.7%)
- **Data Preservation Rate:** **89.3%**

---

## 5. Repository Structure

```
├── Proposed-structure-hackathon.pdf   # Official Deliverable 2 PDF
├── app/
│   ├── build.gradle.kts              # Android Gradle configuration
│   ├── src/main/
│   │   ├── AndroidManifest.xml       # Scoped permissions & :rogue_workload process
│   │   ├── java/org/ransomlab/sentinel/
│   │   │   ├── SentinelApp.kt        # App lifecycle & notification channels
│   │   │   ├── data/model/           # FileEvent, Alert, EvaluationMetrics
│   │   │   ├── detector/             # EntropyCalculator, BehavioralEngine, TelemetryObserver, Mitigation
│   │   │   ├── workloads/            # 2 Benign + 3 Ransomware simulations, WorkloadRunnerService
│   │   │   ├── evaluator/            # LabBenchmarkRunner, TelemetryExporter
│   │   │   └── ui/                   # MainActivity, TelemetryAdapter, EvaluationAdapter
│   │   └── res/                      # Modern dark cyber theme, layouts, vector assets
│   └── src/test/                     # Unit test suite (Entropy, BehavioralEngine, Metrics)
├── docs/
│   └── PSEUDOCODE_SNIPPET.md         # Deliverable 3 pseudocode snippet & mentor briefing
└── scripts/
    ├── setup-toolchain.ps1           # Portable zero-admin JDK17 + Android SDK + AVD installer
    ├── env.ps1                       # Environment PATH loader
    ├── run-adb-lab.ps1               # Automated end-to-end ADB test harness
    ├── run-benign-workloads.ps1      # Standalone benign simulation script
    ├── simulate-in-place-overwrite.ps1# Standalone Sim 1 script
    ├── simulate-rename-encrypt.ps1   # Standalone Sim 2 script
    ├── simulate-wipe-drop.ps1        # Standalone Sim 3 script
    └── generate-hackathon-pdf.js     # Script to generate Proposed-structure-hackathon.pdf
```

---

## 6. How to Run the Demonstration

### Option A: Complete Automated Lab via ADB
To execute the automated end-to-end build, install, and test run:
```powershell
powershell -ExecutionPolicy Bypass -File scripts\run-adb-lab.ps1 -StartEmulator
```

### Option B: Running the Android App Interactively
1. Launch the Android app **ShieldX Lab** on your device or emulator.
2. Tap **"Start Telemetry Monitor"**.
3. Navigate to the **Workloads** tab and trigger any of the benign or ransomware workloads.
4. Watch the real-time event stream in the **Telemetry** tab and inspect the early warning alert banner.
5. In the **Evaluation** tab, tap **"Run Automated Benchmark Suite"** to generate the confusion matrix and export JSON/CSV logs.

### Option C: Running Standalone Simulation Scripts
```powershell
# 1. Run benign workloads
powershell -ExecutionPolicy Bypass -File scripts\run-benign-workloads.ps1 -TargetDir C:\path\to\test_dir

# 2. Run ransomware simulations
powershell -ExecutionPolicy Bypass -File scripts\simulate-in-place-overwrite.ps1 -TargetDir C:\path\to\test_dir
powershell -ExecutionPolicy Bypass -File scripts\simulate-rename-encrypt.ps1 -TargetDir C:\path\to\test_dir
powershell -ExecutionPolicy Bypass -File scripts\simulate-wipe-drop.ps1 -TargetDir C:\path\to\test_dir
```

---

## 7. Deliverable Submission Checklist
- [x] **Private GitHub Repository:** Created and configured.
- [x] **Collaborator Invitation:** Added `aiori-hackathon` (https://github.com/aiori-hackathon) as collaborator.
- [x] **Ownership Transfer:** Ready for transfer to `aiori-hackathon`.
- [x] **Proposed-structure-hackathon.pdf:** Uploaded at repository root.
- [x] **Prototype Demonstration & Pseudocode Snippet:** Documented in [`docs/PSEUDOCODE_SNIPPET.md`](docs/PSEUDOCODE_SNIPPET.md).
- [x] **2 Benign + 3 Ransomware Simulations:** Fully implemented and verified.
- [x] **Empirical Metrics:** Detection delay, files changed, precision, and recall quantified.
