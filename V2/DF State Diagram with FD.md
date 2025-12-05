```mermaid
stateDiagram-v2
    [*] --> S000000

    %% ═══════════════════════════════════════════════════════════
    %% STATE NAMING CONVENTION (6-bit binary)
    %% Bit positions: [1][2][3][4][5][6]
    %% [1] df_event_status
    %% [2] df_restriction_status
    %% [3] enable_climate_snapshot
    %% [4] set_df_conditions
    %% [5] df_conditions_set
    %% [6] set_point_changed_on_device
    %% ═══════════════════════════════════════════════════════════

    %% NOMINAL STATES
    state "**IDLE**<br/>S000000" as S000000
    state "**S1_DONE**<br/>S100000" as S100000
    state "**S2_DONE**<br/>S111000" as S111000
    state "**S3_DONE**<br/>S111100" as S111100
    state "**DF_ACTIVE**<br/>S111110" as S111110
    state "**OVERRIDE**<br/>S111111" as S111111

    %% ACTION STATES
    state "INIT_DF_EVENT<br/>(start_df_event)" as INIT_DF
    state "ENABLE_INTERLOCKS<br/>(df_event_started)" as ENABLE_INT
    state "CAPTURE_SNAPSHOT<br/>(take climate snapshot)" as CAPTURE_SNAP
    state "PARSE_REFERENCE<br/>(Enable DF conditions)" as PARSE_REF
    state "APPLY_SETPOINT" as APPLY_SP
    state "DETECT_OVERRIDE<br/>(thermostat attr Δ)" as DETECT_OVR
    state "HANDLE_OVERRIDE<br/>(Occupant Overrides On Device)" as HANDLE_OVR
    state "RESTORE_SETPOINT<br/>(df_event_ended OR<br/>Occupant Overrides In App)" as RESTORE_SP
    state "RELEASE_INTERLOCKS" as RELEASE_INT

    %% RECOVERY STATES
    state "MANUAL_HA_RESTART" as RESTART
    state "MANUAL_DF_RESET" as RESET
    state "WAIT_CALENDAR_OFF" as WAIT_CAL

    %% FAULT STATES
    state "**FAULT_STUCK**<br/>Partial boolean state" as FAULT_STUCK

    %% ═══════════════════════════════════════════════════════════
    %% STAGE 1: IDLE → S1_DONE
    %% ═══════════════════════════════════════════════════════════
    S000000 --> INIT_DF: calendar OFF→ON
    INIT_DF --> S100000: ✓ SUCCESS
    INIT_DF --> S000000: F01 Plant unavailable<br/>F02 Invalid HVAC mode
    INIT_DF --> RESTART: F03 Actuator service fail
    INIT_DF --> RESET: F04 Partial boolean fail

    %% ═══════════════════════════════════════════════════════════
    %% STAGE 2: S1_DONE → S2_DONE
    %% ═══════════════════════════════════════════════════════════
    S100000 --> ENABLE_INT: df_event_status OFF→ON
    ENABLE_INT --> S111000: ✓ SUCCESS
    ENABLE_INT --> RESET: F05 Actuator service fail

    %% ═══════════════════════════════════════════════════════════
    %% STAGE 3: S2_DONE → S3_DONE
    %% ═══════════════════════════════════════════════════════════
    S111000 --> CAPTURE_SNAP: enable_climate_snapshot OFF→ON
    CAPTURE_SNAP --> S111100: ✓ SUCCESS
    CAPTURE_SNAP --> RESET: F06 Guard condition fail
    CAPTURE_SNAP --> S111100: F07 Sensor NULL (silent)
    CAPTURE_SNAP --> RESET: F08 Memory write fail

    %% ═══════════════════════════════════════════════════════════
    %% STAGE 4: S3_DONE → DF_ACTIVE
    %% ═══════════════════════════════════════════════════════════
    S111100 --> PARSE_REF: set_df_conditions OFF→ON
    PARSE_REF --> APPLY_SP: ✓ Reference valid
    PARSE_REF --> WAIT_CAL: F09 JSON parse error<br/>F10 Missing T_delta/season<br/>F11 Season ≠ HVAC mode
    APPLY_SP --> S111110: ✓ SUCCESS
    APPLY_SP --> RESET: F12 Plant actuator fail<br/>F13 Setpoint out of range
    WAIT_CAL --> RESTORE_SP: calendar ON→OFF

    %% ═══════════════════════════════════════════════════════════
    %% STAGE 5: DF_ACTIVE (Steady State)
    %% ═══════════════════════════════════════════════════════════
    S111110 --> DETECT_OVR: thermostat attr changes
    DETECT_OVR --> S111110: ✓ No override (normal fluctuation)
    DETECT_OVR --> HANDLE_OVR: ✓ Override confirmed
    DETECT_OVR --> FAULT_STUCK: F14 False positive

    %% ═══════════════════════════════════════════════════════════
    %% STAGE 6: Override Handling
    %% ═══════════════════════════════════════════════════════════
    HANDLE_OVR --> S111111: set_point_changed ON
    S111111 --> RELEASE_INT: Operator takeover
    RELEASE_INT --> S000000: ✓ SUCCESS
    RELEASE_INT --> RESET: F15 Partial release

    %% ═══════════════════════════════════════════════════════════
    %% STAGE 7: Normal Termination
    %% ═══════════════════════════════════════════════════════════
    S111110 --> RESTORE_SP: calendar ON→OFF<br/>OR df_restriction ON→OFF
    RESTORE_SP --> RELEASE_INT: ✓ SUCCESS
    RESTORE_SP --> FAULT_STUCK: F16 Snapshot corrupted<br/>F17 Restore service fail

    %% ═══════════════════════════════════════════════════════════
    %% RECOVERY PATHS
    %% ═══════════════════════════════════════════════════════════
    RESTART --> S000000: After HA restart
    RESET --> S000000: script.reset_df_state
    FAULT_STUCK --> RESET: Manual intervention

    %% ═══════════════════════════════════════════════════════════
    %% NOTES
    %% ═══════════════════════════════════════════════════════════
    note right of S000000
        All booleans OFF
        Plant: Nominal operation
        Controller: Standby
    end note

    note right of S111110
        DF setpoints applied
        Monitoring for:
        • calendar OFF
        • df_restriction OFF
        • thermostat attr Δ
    end note

    note left of FAULT_STUCK
```
        Inconsistent boolean state
        Requires manual reset
    end note
