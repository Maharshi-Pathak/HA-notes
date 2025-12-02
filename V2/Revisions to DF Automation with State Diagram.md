# DF Automation Control System - Fault Reference

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        SUPERVISORY CONTROLLER                        │
│                      (Home Assistant Automation)                     │
├─────────────────────────────────────────────────────────────────────┤
│  REFERENCE INPUT          CONTROLLER              PLANT OUTPUT       │
│  ┌─────────────┐         ┌─────────┐            ┌─────────────┐     │
│  │  Calendar   │─────────│  State  │────────────│ Thermostat  │     │
│  │  (T_delta,  │  r(t)   │ Machine │   u(t)     │   (HVAC)    │     │
│  │   season)   │         │         │            │             │     │
│  └─────────────┘         └────┬────┘            └──────┬──────┘     │
│                               │                        │            │
│                               │    ┌──────────┐       │            │
│                               │    │ FEEDBACK │       │ y(t)       │
│                               └────│  MONITOR │◄──────┘            │
│                                    │ (Override│                     │
│                                    │  Detect) │                     │
│                                    └──────────┘                     │
│  INTERLOCK FLAGS (State Memory):                                    │
│  [df_event_status, df_restriction_status, enable_climate_snapshot,  │
│   set_df_conditions, df_conditions_set, df_applying_setpoint,       │
│   set_point_changed_on_device]                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Fault Classification by Subsystem

### INITIALIZATION SUBSYSTEM

| Fault Code | Fault Type | Trigger | Cause | Interlock State | Recovery |
|------------|------------|---------|-------|-----------------|----------|
| **FAULT_01** | Sensor Fault | S1: EVENT_DETECT | Plant entity unavailable | All OFF | Auto (no transition) |
| **FAULT_02** | Mode Fault | S1: EVENT_DETECT | Plant in invalid mode (off/fan_only) | All OFF | Auto (no transition) |
| **FAULT_03** | Actuator Fault | S1: EVENT_DETECT | `input_boolean.turn_on` fails | All OFF | Manual HA restart |
| **FAULT_04** | Actuator Fault | S2: INTERLOCK_SET | Multiple boolean service fails | Partial ON | Manual reset |

### SNAPSHOT ACQUISITION SUBSYSTEM

| Fault Code | Fault Type | Trigger | Cause | Interlock State | Recovery |
|------------|------------|---------|-------|-----------------|----------|
| **FAULT_05** | Guard Fault | S3: STATE_CAPTURE | `df_event_status` externally cleared | Partial ON | Manual reset |
| **FAULT_06** | Sensor Fault | S3: STATE_CAPTURE | Plant attribute returns NULL | Silent (propagates) | None until FAULT_19 |
| **FAULT_07** | Memory Fault | S3: STATE_CAPTURE | `input_text.set_value` fails | 3 flags ON | Manual reset |

### SETPOINT OVERRIDE SUBSYSTEM

| Fault Code | Fault Type | Trigger | Cause | Interlock State | Recovery |
|------------|------------|---------|-------|-----------------|----------|
| **FAULT_08** | Reference Fault | S4a: REFERENCE_PARSE | `description` null/empty/malformed JSON | 4 flags ON | Wait for calendar OFF |
| **FAULT_09** | Reference Fault | S4a: REFERENCE_PARSE | Missing `T_delta` or `season` keys | 4 flags ON | Wait for calendar OFF |
| **FAULT_10** | Mode Fault | S4a: REFERENCE_PARSE | Season ≠ HVAC mode (no branch match) | 4 flags ON | Wait for calendar OFF |
| **FAULT_11** | Guard Fault | S4b: ACTUATOR_GUARD_ON | Guard boolean service fails | 4 flags ON | Manual reset |
| **FAULT_12** | Plant Fault | S4c: SETPOINT_WRITE | `climate.set_temperature` fails | Guard ON, 4 flags ON | Manual reset |
| **FAULT_13** | Range Fault | S4c: SETPOINT_WRITE | Calculated SP outside plant limits | Guard ON, 4 flags ON | Manual intervention |
| **FAULT_14** | Guard Fault | S4d: ACTUATOR_GUARD_OFF | Guard stuck ON after timeout | All 5 flags ON, Guard ON | **CRITICAL** - Manual |

### FEEDBACK MONITOR SUBSYSTEM

| Fault Code | Fault Type | Trigger | Cause | Interlock State | Recovery |
|------------|------------|---------|-------|-----------------|----------|
| **FAULT_15** | Guard Fault | S6: DEVIATION_DETECT | Guard stuck from FAULT_14 | Guard ON | Override missed |
| **FAULT_16** | Detection Fault | S6: DEVIATION_DETECT | Race condition (guard OFF during write) | Override flag ON | False positive → S7 |

### MANUAL OVERRIDE SUBSYSTEM

| Fault Code | Fault Type | Trigger | Cause | Interlock State | Recovery |
|------------|------------|---------|-------|-----------------|----------|
| **FAULT_17** | Actuator Fault | S7: OPERATOR_TAKEOVER | Partial interlock release | Mixed ON/OFF | Manual reset |

### RECOVERY SUBSYSTEM

| Fault Code | Fault Type | Trigger | Cause | Interlock State | Recovery |
|------------|------------|---------|-------|-----------------|----------|
| **FAULT_18** | Guard Fault | S8a: RESTORE_GUARD_ON | Condition check fails | All flags ON | Manual reset |
| **FAULT_19** | Memory Fault | S8b: SETPOINT_RESTORE | Snapshot contains NULL/"None"/NaN | Guard ON | **CRITICAL** - Plant damage |
| **FAULT_20** | Plant Fault | S8b: SETPOINT_RESTORE | `climate.set_temperature` or `ecobee.resume` fails | Guard ON | Manual intervention |
| **FAULT_21** | Guard Fault | S8c: RESTORE_GUARD_OFF | Guard stuck ON after restore | Guard ON, flags ON | Manual reset |
| **FAULT_22** | Actuator Fault | S8d: INTERLOCK_RELEASE | Partial flag reset | Mixed ON/OFF | Manual reset |

---

## Interlock State Vector

| State | df_event | df_restriction | enable_snapshot | set_df_cond | df_cond_set | applying_sp | sp_changed |
|-------|:--------:|:--------------:|:---------------:|:-----------:|:-----------:|:-----------:|:----------:|
| **IDLE** | 0 | 0 | 0 | 0 | 0 | 0 | 0 |
| **S1 Complete** | 1 | 0 | 0 | 0 | 0 | 0 | 0 |
| **S2 Complete** | 1 | 1 | 1 | 0 | 0 | 0 | 0 |
| **S3 Complete** | 1 | 1 | 1 | 1 | 0 | 0 | 0 |
| **S4b (Guard ON)** | 1 | 1 | 1 | 1 | 0 | **1** | 0 |
| **S4d Complete** | 1 | 1 | 1 | 1 | 1 | 0 | 0 |
| **S5 Steady** | 1 | 1 | 1 | 1 | 1 | 0 | 0 |
| **S6 Override** | 1 | 1 | 1 | 1 | 1 | 0 | **1** |
| **S8a (Guard ON)** | 1 | 1 | 1 | 1 | 1 | **1** | 0 |
| **Recovery Done** | 0 | 0 | 0 | 0 | 0 | 0 | 0 |

---

## Fault Severity Classification

| Severity | Faults | Impact | MTTR |
|----------|--------|--------|------|
| 🔴 **CRITICAL** | FAULT_14, FAULT_19 | Plant damage possible, feedback loop broken | Immediate manual intervention |
| 🟠 **MAJOR** | FAULT_08-10, FAULT_15-16 | DF event fails silently, user confusion | Wait for calendar cycle or manual |
| 🟡 **MINOR** | FAULT_03-07, FAULT_17-18, FAULT_20-22 | Stuck state, requires manual reset | Run reset script |
| 🟢 **BENIGN** | FAULT_01, FAULT_02 | Auto-recovery, no state change | None required |

---

## Fault Detection Logic

```
FAULT DETECTOR (run every state transition):
─────────────────────────────────────────────
IF (df_event_status == ON) AND (calendar == OFF) AND (elapsed > 4h):
    → FAULT: Stale event, interlocks not released
    
IF (df_applying_setpoint == ON) AND (elapsed > 60s):
    → FAULT_14/21: Guard timeout exceeded
    
IF (df_conditions_set == ON) AND (df_event_status == OFF):
    → FAULT: Inconsistent interlock state
    
IF (snapshot_temp == "None") OR (snapshot_temp == ""):
    → FAULT_06: Sensor NULL detected (pre-emptive)
```

---

## Recovery Procedure

### Automated Recovery Script

```yaml
automation:
  - alias: "DF Fault Detector & Auto-Recovery"
    trigger:
      - platform: state
        entity_id: input_boolean.df_applying_setpoint
        to: "on"
        for:
          seconds: 60
    condition:
      - condition: state
        entity_id: input_boolean.df_conditions_set
        state: "on"
    action:
      - service: persistent_notification.create
        data:
          title: "⚠️ DF FAULT DETECTED"
          message: "Guard timeout - initiating recovery"
      - service: script.reset_df_state
```

### Manual Reset Script

```yaml
script:
  reset_df_state:
    alias: "DF Controller - Manual Reset"
    sequence:
      - service: input_boolean.turn_off
        target:
          entity_id:
            - input_boolean.df_event_status
            - input_boolean.df_restriction_status
            - input_boolean.enable_climate_snapshot
            - input_boolean.set_df_conditions
            - input_boolean.df_conditions_set
            - input_boolean.df_applying_setpoint
            - input_boolean.set_point_changed_on_device
      - service: persistent_notification.create
        data:
          title: "DF Controller Reset"
          message: "All interlocks released at {{ now() }}"
```

---

## Control System Terminology Reference

| Term | DF Automation Equivalent |
|------|--------------------------|
| **Plant** | Thermostat / HVAC system |
| **Controller** | Home Assistant automation |
| **Reference Input r(t)** | Calendar event (T_delta, season) |
| **Control Output u(t)** | `climate.set_temperature` command |
| **Plant Output y(t)** | Thermostat attribute states |
| **Feedback** | Override detection (S6) |
| **Interlock** | Boolean flags preventing unsafe transitions |
| **Guard** | `df_applying_setpoint` (inhibits feedback during command) |
| **Setpoint SP** | Target temperature |
| **Fault** | Error condition requiring intervention |
| **MTTR** | Mean Time To Recovery |
