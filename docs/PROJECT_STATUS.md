# Hybrid Detection System — Project Status

## Project Overview
**Course:** CCCY 421 — Senior Project 2  
**University:** University of Jeddah, College of Computer Sciences and Engineering, Cybersecurity Department  
**Supervisor:** Dr. Naif Alzahrani  
**Status:** ✅ Presentation delivered, awaiting grade  
**Next step:** Publish to GitHub

---

## Team Members & Contributions

| Student | Role |
|---|---|
| Abdullah Asaad Ladjal | AI Agent (Llama 3.2, Ollama, explain.py) + Main Pipeline (pipeline.py) |
| Tal Mohammed Faden | Fusion Layer (R1–R12) + Wazuh SIEM Integration |
| Ahmed Khalid Humadi | Suricata Rules (12 custom LOCAL) + MITRE ATT&CK Mapping (3-tier) |
| Khalid Ali AL-Mufarrih | System Architecture + Lab Environment + Testing & Validation |
| Abdulmajeed Meshal Almalki | ML Training + Dataset Collection + Model Comparison (6 models) |

---

## Lab Environment

| VM | OS | IP | Role | RAM |
|---|---|---|---|---|
| Kali Linux | Kali 2024 | 10.0.0.5 | Attacker | 4GB |
| Ubuntu Desktop | Ubuntu 22.04 | 10.0.0.6 | Victim | 4GB |
| IDS/SIEM VM | Ubuntu Server | 10.0.0.4 | Detection | 12GB |
| Windows Host | Windows 11 | 192.168.1.74 | Ollama GPU | 32GB |

**Network:** VirtualBox NAT Network 10.0.0.0/24, promiscuous mode on enp0s3  
**Ollama:** llama3.2:3b on host GPU, port 11434, OLLAMA_HOST=0.0.0.0

---

## System Components

### 1. Suricata NIDS ✅
- Version: 7.0.10
- Interface: enp0s3 (promiscuous mode)
- Rules: ET/Open (30,000+) + 12 custom LOCAL rules
- Output: /var/log/suricata/eve.json (with MITRE metadata)
- Custom rules cover: SYN/NULL/FIN/XMAS scans, SSH/FTP/HTTP brute force, Hydra, Nikto, dirb

### 2. ML Anomaly Detection ✅
- Tool: NFStream 6.5.3 (65 bidirectional flow features)
- Model: LightGBM (best_model_custom.pkl) — selected for lowest FPR
- Feature names: feature_names_custom.pkl (65 features)
- Training: Python 3.13, venv at ~/ml-ids/venv

**Custom Dataset (live pipeline):**
- 5,000 benign flows + 5,000 attack flows = 10,000 total
- 80/20 stratified split
- Accuracy: 95.30% | FPR: 0.44%

**Combined Benchmark Dataset (academic validation):**
- CICIDS2017 (400K rows) + UNSW-NB15 (700K rows) + custom = 1,110,001 total
- Balanced to 120,504 samples, 29 clean features
- XGBoost: 99.55% accuracy

**Six models compared:**
| Model | Test Acc | F1 | FPR |
|---|---|---|---|
| LightGBM ★ | 0.9530 | 0.9509 | 0.0044 |
| XGBoost | 0.9520 | 0.9501 | 0.0098 |
| MLP | 0.9515 | 0.9493 | 0.0066 |
| AdaBoost | 0.9490 | 0.9464 | 0.0033 |
| RandomForest | 0.9425 | 0.9405 | 0.0257 ⚠ overfit |
| DecisionTree | 0.9250 | 0.9251 | 0.0767 ⚠ overfit |

### 3. Fusion Layer ✅
- File: ~/ml-ids/fusion.py
- 12 rules (R1–R12) mapping (S_alert, S_sev, anomaly_score) → L0–L4
- Alert cooldown: 60s per (src_ip, attack_category)
- Suricata-confirmed alerts bypass cooldown

| Level | Confidence | Action |
|---|---|---|
| L0 | Benign | No action |
| L1 | Low | Log only |
| L2 | Suspicious | Manual review |
| L3 | High | Immediate investigate |
| L4 | Confirmed | Immediate response |

### 4. MITRE ATT&CK Mapping ✅
- File: ~/ml-ids/mitre.py
- Database: enterprise-attack.json (v14, 691 techniques)
- Library: mitreattack-python

**Three-tier priority:**
1. Suricata ET rule embedded metadata (authoritative) → `suricata_metadata`
2. Signature text search in MITRE STIX database → `mitre_db_lookup`
3. Port/behavior-based fallback → `port_fallback`

### 5. AI Agent ✅
- Model: Llama 3.2 3B via Ollama on host GPU
- URL: http://192.168.1.74:11434/api/generate
- Response time: ~5.6 seconds
- Design: async background thread (non-blocking)
- Triggers: L3/L4 alerts only
- Files: pipeline.py (inline), explain.py (on-demand)

### 6. Wazuh SIEM Integration ✅
- Version: 4.7.5 bare-metal (NOT Docker)
- Custom decoder: /var/ossec/etc/decoders/hybrid_ids_decoder.xml
- Custom rules: /var/ossec/etc/rules/hybrid_ids_rules.xml
  - 100001: L4 → Wazuh level 10
  - 100002: L3 → Wazuh level 7
  - 100003: L2 → Wazuh level 5
  - 100004: L1 → Wazuh level 3
  - 100005: L0 → Wazuh level 1
- Log source: /var/ossec/logs/active-responses.log (JSON)

---

## Key Files

```
~/ml-ids/
├── pipeline.py              # Main detection pipeline (run with sudo venv/bin/python3)
├── explain.py               # On-demand AI analyst tool
├── fusion.py                # R1-R12 fusion rules
├── mitre.py                 # MITRE ATT&CK mapping (3-tier)
├── train_custom.py          # Train on custom NFStream data
├── train_combined.py        # Train on combined benchmark datasets
├── build_combined_dataset.py
├── test_combined.py         # Evaluate model on unseen data
├── best_model_custom.pkl    # ACTIVE pipeline model (LightGBM, 65 features)
├── best_model_120k.pkl      # Benchmark model (XGBoost, 29 features)
├── best_model_combined.pkl  # Combined model (XGBoost, 29 features)
├── feature_names_custom.pkl # 65 features
├── feature_names_120k.pkl   # 29 features
├── enterprise-attack.json   # MITRE ATT&CK v14 STIX database
├── benign_flows.csv         # 5,000 benign flows
├── attack_flows.csv         # 5,000 attack flows
├── UNSW-NB15_1.csv          # 700,001 rows
├── archive/                 # CICIDS2017 CSVs (8 files)
└── combined_dataset.pkl     # 120,504 balanced samples

/var/log/suricata/
├── eve.json                 # Suricata alerts (JSON, tailed by pipeline)
└── fast.log                 # Human-readable alerts

/var/ossec/
├── logs/active-responses.log    # Enriched pipeline alerts → Wazuh reads this
├── logs/alerts/alerts.json      # Wazuh processed alerts
├── etc/rules/hybrid_ids_rules.xml
└── etc/decoders/hybrid_ids_decoder.xml

/var/lib/suricata/rules/
└── local.rules              # 12 custom LOCAL rules
```

---

## Running the Pipeline

```bash
# On IDS VM (10.0.0.4)
cd ~/ml-ids
source venv/bin/activate
sudo venv/bin/python3 pipeline.py
```

**Startup output:**
```
[*] Hybrid Detection Pipeline started
    ML Model   : LGBMClassifier
    AI Model   : llama3.2:3b (async)
    Victim IP  : 10.0.0.6
    Cooldown   : 60s per attack category
    MITRE      : Suricata metadata > DB lookup > port fallback
```

---

## Live Testing Results

### Full Attack Session
| Metric | Value |
|---|---|
| Total alerts | 38,516 |
| L4 Confirmed | 9,397 (24.4%) |
| L3 High confidence | 7,820 (20.3%) |
| L2 Suspicious | 8,960 (23.3%) |
| L1 Low | 1,511 (3.9%) |
| L0 Benign | 10,828 (28.1%) |
| ML only detections | 9,814 (72.3%) |
| Both ML + Suricata | 3,441 (25.4%) |
| Suricata only | 654 (4.8%) |

### KEY RESULT — Stealth Scan
```
nmap -sS -T0 --scan-delay 5s 10.0.0.6
Suricata:  0 alerts
ML module: 18 alerts — all L4 — score=0.9999
```

### Fresh Traffic Generalization (1,284 unseen flows)
- Accuracy: 87.15%
- FPR: 1.7% (2 false positives out of 116 benign)
- Precision: 0.9980

---

## Attack Categories Detected

| Category | Count |
|---|---|
| ANOMALY | 31,052 |
| PORT_SCAN | 6,030 |
| WEB_ATTACK | 897 |
| SURICATA_ALERT | 251 |
| EXPLOIT | 217 |
| BRUTE_FORCE | 49 |
| SSH_ATTACK | 18 |
| SMB_ATTACK | 2 |

---

## Documents Produced

| Document | File | Status |
|---|---|---|
| Final Report | hybrid_ids_report_v2.docx | ✅ Done |
| Presentation | hybrid_ids_presentation.pptx | ✅ Done (14 slides) |
| Poster | poster_formatted.pptx | ✅ Done (formatted per Dr. Malik requirements) |

---

## Presentation Structure

| Slides | Speaker | Topics |
|---|---|---|
| 1–3 | Khalid | Title, Problem & Objectives, Tools & Technologies |
| 4–6 | Ahmed | Architecture, Fusion Layer, Datasets & Experimentation |
| 7–10 | Abdulmajeed | ML Results, Live Detection, Key Finding, Assessment |
| 11–14 | Abdullah | Conclusion, Limitations, Future Work, Contributions |
| Demo | Abdullah | Live pipeline demo on 3 machines |

---

## Pending / Next Steps

- [ ] **Publish to GitHub** — next task
- [ ] Add Turnitin certificate to report when available
- [ ] Optional: upgrade AI agent to fully autonomous (tool-use, memory, AbuseIPDB)

---

## Known Issues / Notes

- Docker caused ossec.conf XML corruption — Wazuh must be bare-metal
- Combined 120K model detects only 21 live alerts vs 38,516 for custom model (domain mismatch)
- OLLAMA_HOST=0.0.0.0 must be set as system environment variable on Windows host
- Whitelist ports: {1514, 1515, 55000} (Wazuh agent communication)
- Tal cannot speak during presentation

---

## Q&A Prep

| Question | Answer |
|---|---|
| Why only 10,000 custom samples? | Domain mismatch: 120K benchmark model detected 21 live alerts vs 38,516 for custom. Data must match deployment environment. |
| What is FPR? | 0.44% — only 4 false positives per 1,000 benign flows. |
| How does Fusion Layer reduce FP? | Requires corroboration — low ML + low Suricata = L1 not L4. |
| Why Llama not GPT? | Privacy, no API costs, runs locally on lab GPU. |
| Difference from just Suricata? | 72.3% of attacks would be missed. Stealth scan: 0 Suricata alerts, 18 ML L4 detections. |
| How many CICIDS2017 features? | Original: 79 (including label). We used 29 clean features after removing those with >80% zeros due to domain mismatch. |
