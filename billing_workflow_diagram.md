# Low-Level Billing Workflow

```mermaid
flowchart TD
    A[Start: Receive Request<br/>(user_id, upload_dir)] --> B[Extract Audio Paths<br/>(audio_paths list)]
    B --> C[Fetch User Data]

    C -->|Java API| C1[GET /fetch-user<br/>{user_id}]
    C -->|Fallback DB| C2[SELECT subscription_plan,<br/>credits_available<br/>FROM user_credentials+user_credits]

    C --> D{Subscription Active?}
    D -->|No| D1[Java API POST /notify-subscription<br/>→ Raise Exception<br/>→ HALT]
    D -->|Yes| E[Get Billing Config<br/>(cost_per_minute, max_files)]

    E --> F[Calculate Costs Per File]
    F --> F1[For each audio_file:<br/>Get Duration (ffprobe/librosa)]
    F1 --> F2[Cost = (duration / 60) * rate_per_minute]
    F2 --> F3[Sum total_cost = Σ file costs]

    F3 --> G{Plan Limit Exceeded?}
    G -->|Yes| G1[Raise Exception: Plan Limit Exceeded<br/>→ HALT]
    G -->|No| H{Credits >= total_cost?}

    H -->|Yes| H1[Set process_files = all audio_paths]
    H -->|No| I[Calculate Affordable Subset]

    I --> I1[Query DB: unprocessed files<br/>(recording_flag = False)]
    I1 --> I2[Cumulative add until cum_cost <= credits]
    I2 --> J{max_files == 0?}

    J -->|Yes| J1[Java API POST /notify-recharge<br/>→ HALT]
    J -->|No| K[Java API POST /ask-partial]

    K -->|No| K1[Java API POST /notify-recharge<br/>→ HALT]
    K -->|Yes| K2[Set process_files = selected[:max_files]]

    H1 --> L[Process Files (process_recording_data)]
    K2 --> L

    L --> L1[Parallel ThreadPoolExecutor]
    L1 --> L2[Per File: Validate File Exists,<br/>Get DB Details]
    L2 --> L3[Apply RateLimiter]
    L3 --> L4[Transcribe (ASR)]
    L4 --> L5[Analyze (AI models)]
    L5 --> L6[Insert Results into DB]
    L6 --> L7[Insert Token Usage DB]
    L7 --> L8[Mark File Done<br/>(recording_flag=True)]
    L8 --> L9[Move File to Completed Dir]

    L9 --> M[After All Done:<br/>Calc actual_cost = Σ processed file costs]

    M --> N[Update Credits (Atomic Transaction)]
    N --> N1[UPDATE user_credits -= actual_cost]
    N --> N2[INSERT billing_logs (deduct event)]
    N --> N3[COMMIT / ROLLBACK]

    N3 --> O[End: Return Success<br/>(status: ok, processed_files count)]
    D1 --> O
    G1 --> O
    J1 --> O
    K1 --> O

    classDef error fill=#ffcccc,stroke=#ff0000,color=#000
    classDef process fill=#e0f7fa,stroke=#006064,color=#000
    classDef decision fill=#fff3e0,stroke=#ef6c00,color=#000
    classDef db fill=#ede7f6,stroke=#4527a0,color=#000

    class A,B,C,E,F,F1,F2,F3,I,I1,I2,L,L1,L2,L3,L4,L5,L6,L7,L8,L9,M,N,N1,N2,N3,O process
    class D,G,H,J,K decision
    class C2,I1,N1,N2 db
    class D1,G1,J1,K1 error
```

