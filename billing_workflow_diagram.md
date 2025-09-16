flowchart TD
    A[Start: Receive Request (user_id, upload_dir)] --> B[Extract Audio Paths (audio_paths list)]
    B --> C[Fetch User Data]

    C -->|Java API| C1[GET /fetch-user (user_id)]
    C -->|Fallback DB| C2[SELECT subscription_plan, credits_available FROM user_credentials and user_credits]

    C --> D{Subscription Active?}
    D -->|No| D1[Java API POST /notify-subscription - Raise Exception - HALT]
    D -->|Yes| E[Get Billing Config (cost_per_minute, max_files)]

    E --> F[Calculate Costs Per File]
    F --> F1[For each audio_file: Get Duration (ffprobe or librosa)]
    F1 --> F2[Cost = (duration / 60) * rate_per_minute]
    F2 --> F3[Sum total_cost = SUM file costs]

    F3 --> G{Plan Limit Exceeded?}
    G -->|Yes| G1[Raise Exception - Plan Limit Exceeded - HALT]
    G -->|No| H{Credits >= total_cost?}

    H -->|Yes| H1[Set process_files = all audio_paths]
    H -->|No| I[Calculate Affordable Subset]

    I --> I1[Query DB: unprocessed files where recording_flag = False]
    I1 --> I2[Cumulative add until cum_cost <= credits]
    I2 --> J{max_files == 0?}

    J -->|Yes| J1[Java API POST /notify-recharge - HALT]
    J -->|No| K[Java API POST /ask-partial]

    K -->|No| K1[Java API POST /notify-recharge - HALT]
    K -->|Yes| K2[Set process_files = selected upto max_files]

    H1 --> L[Process Files (process_recording_data)]
    K2 --> L

    L --> L1[Parallel ThreadPoolExecutor]
    L1 --> L2[Per File: Validate File Exists, Get DB Details]
    L2 --> L3[Apply RateLimiter]
    L3 --> L4[Transcribe (ASR)]
    L4 --> L5[Analyze (AI models)]
    L5 --> L6[Insert Results into DB]
    L6 --> L7[Insert Token Usage DB]
    L7 --> L8[Mark File Done (recording_flag=True)]
    L8 --> L9[Move File to Completed Dir]

    L9 --> M[After All Done: Calculate actual_cost = SUM processed file costs]

    M --> N[Update Credits (Atomic Transaction)]
    N --> N1[UPDATE user_credits -= actual_cost]
    N --> N2[INSERT billing_logs (deduct event)]
    N --> N3[COMMIT or ROLLBACK]

    N3 --> O[End: Return Success (status: ok, processed_files count)]
    D1 --> O
    G1 --> O
    J1 --> O
    K1 --> O
