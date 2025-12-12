```mermaid
stateDiagram-v2
    [*] --> IDLE
    
    state "IDLE" as IDLE
    state "DF EVENT ACTIVE" as DF_ACTIVE {
        state "Initialization" as INIT
        state "Snapshot Acquisition" as SNAP
        state "Setpoint Override" as CTRL
        state "Steady State" as STEADY
        
        [*] --> INIT
        INIT --> SNAP
        SNAP --> CTRL
        CTRL --> STEADY
    }

    state "OVERRIDE DETECTED" as OVERRIDE {
        state "Feedback Monitor" as FB
        state "Manual Override" as MAN
        
        [*] --> FB
        FB --> MAN
    }

    state "RESTORATION" as RESTORE {
        state "Recovery" as RECV
        [*] --> RECV
    }

    IDLE --> DF_ACTIVE: Calendar ON
    DF_ACTIVE --> OVERRIDE: Plant deviation detected
    DF_ACTIVE --> RESTORE: Calendar OFF
    OVERRIDE --> IDLE: Operator takeover
    RESTORE --> IDLE: Recovery complete
```
┌─────────────────────────────────────────────────────────────────────┐
│                        SUPERVISORY CONTROLLER                       │
│                      (Home Assistant Automation)                    │
├─────────────────────────────────────────────────────────────────────┤
│  REFERENCE INPUT          CONTROLLER              PLANT OUTPUT      │
│  ┌─────────────┐         ┌─────────┐            ┌─────────────┐     │
│  │  Calendar   │─────────│  State  │────────────│ Thermostat  │     │
│  │  (T_delta,  │  r(t)   │ Machine │   u(t)     │   (HVAC)    │     │
│  │   season)   │         │         │            │             │     │
│  └─────────────┘         └────┬────┘            └──────┬──────┘     │
│                               │                        │            │
│                               │    ┌──────────┐        │            │
│                               │    │ FEEDBACK │        │ y(t)       │
│                               └────│  MONITOR │◄────── ┘            │
│                                    │ (Override│                     │
│                                    │  Detect) │                     │
│                                    └──────────┘                     │
│  INTERLOCK FLAGS (State Memory):                                    │
│  [df_event_status, df_restriction_status, enable_climate_snapshot,  │
│   set_df_conditions, df_conditions_set, df_applying_setpoint,       │
│   set_point_changed_on_device]                                      │
└─────────────────────────────────────────────────────────────────────┘

```mermaid
stateDiagram-v2
    [*] --> S000000

    %% NOMINAL STATES
    state "**IDLE**<br/>S000000" as S000000
    state "**S1_DONE**<br/>S100000" as S100000
    state "**S2_DONE**<br/>S111000" as S111000
    state "**S3_DONE**<br/>S111100" as S111100
    state "**DF_ACTIVE**<br/>S111110" as S111110
    state "**OVERRIDE**<br/>S111111" as S111111

    %% TRANSITIONS
    S000000 --> S100000: calendar OFF→ON
    S100000 --> S111000: df_event_status OFF→ON
    S111000 --> S111100: enable_climate_snapshot OFF→ON
    S111100 --> S111110: set_df_conditions OFF→ON
    S111110 --> S111111: thermostat attr changes (Override)
    S111111 --> S000000: Operator takeover (Reset)
    S111110 --> S000000: Normal Termination (Restore)
```

```mermaid
stateDiagram-v2
    [*] --> IDLE

    state "DF EVENT ACTIVE" as DF_ACTIVE {
        state "INITIALIZATION SUBSYSTEM" as INIT_SUB {
            state "S1: EVENT_DETECT" as S1
            state "S2: INTERLOCK_SET" as S2
        }

        state "SNAPSHOT ACQUISITION" as SNAP_SUB {
            state "S3: STATE_CAPTURE" as S3
        }

        state "SETPOINT OVERRIDE" as CTRL_SUB {
            state "S4a: REFERENCE_PARSE" as S4a
            state "S4b: ACTUATOR_GUARD_ON" as S4b
            state "S4c: SETPOINT_WRITE" as S4c
            state "S4d: ACTUATOR_GUARD_OFF" as S4d

            [*] --> S4a
            S4a --> S4b: Reference valid
            S4b --> S4c: Guard engaged
            S4c --> S4d: Plant acknowledged
            S4d --> [*]: Control active
        }

        state "STEADY STATE" as STEADY_SUB {
            state "S5: HOLD_SETPOINT" as S5
        }

        [*] --> INIT_SUB
        INIT_SUB --> SNAP_SUB: Init complete
        SNAP_SUB --> CTRL_SUB: Snapshot acquired
        CTRL_SUB --> STEADY_SUB: Control engaged
    }

    IDLE --> DF_ACTIVE: Calendar ON
    DF_ACTIVE --> IDLE: Calendar OFF / Override
```
