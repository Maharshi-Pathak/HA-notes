# Demand Flexibility Automation Blueprint: Analysis, Debugging, and Deployment Guide

## Document Overview

This document summarizes a comprehensive analysis of a Home Assistant automation blueprint designed to manage Demand Flexibility (DF) events for thermostats. The analysis covers error identification, state machine modeling, dashboard development, and deployment planning for multi-instance rollouts.

**Document Version:** 1.0  
**Date:** December 2024  
**Scope:** Home Assistant DF Automation Blueprint

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Original Blueprint Analysis](#2-original-blueprint-analysis)
3. [Critical Issues Identified](#3-critical-issues-identified)
4. [State Machine Model](#4-state-machine-model)
5. [Fault Analysis](#5-fault-analysis)
6. [Guard Signal Mechanism](#6-guard-signal-mechanism)
7. [Monitoring Dashboard](#7-monitoring-dashboard)
8. [Deployment Task List](#8-deployment-task-list)
9. [Appendices](#9-appendices)

---

## 1. Introduction

### 1.1 Purpose

The Demand Flexibility (DF) automation manages thermostat setpoint adjustments during utility demand response events. When a DF event is signaled via a calendar entity, the automation captures the current thermostat state (snapshot), applies a temperature offset (T_delta), maintains the adjusted setpoint during the event, and restores the original state when the event ends or the occupant overrides.

### 1.2 Key Components

| Component | Entity Type | Purpose |
|-----------|-------------|---------|
| Calendar | `calendar.*` | Triggers DF events, contains T_delta and season in JSON description |
| Thermostat | `climate.*` | The controlled HVAC plant |
| State Flags | `input_boolean.*` | Track automation state machine progression |
| Snapshot Storage | `input_text.*` | Store pre-DF thermostat settings for restoration |

### 1.3 Operational Modes

The automation operates in four high-level modes:

1. **IDLE** - Awaiting calendar trigger, all booleans OFF
2. **DF EVENT ACTIVE** - Event in progress, setpoints adjusted
3. **OVERRIDE DETECTED** - Occupant intervention detected
4. **RESTORATION** - Returning plant to pre-DF state

---

## 2. Original Blueprint Analysis

### 2.1 Blueprint Structure

The original blueprint uses a multi-stage state machine controlled by six `input_boolean` entities:

```yaml
# State tracking booleans (original automation)
input_boolean:
  df_event_status            # Master DF event flag
  df_restriction_status      # UI restriction active
  enable_climate_snapshot    # Snapshot capture enabled
  set_df_conditions          # DF setpoint application enabled
  df_conditions_set          # DF setpoints successfully applied
  set_point_changed_on_device # Occupant override detected
```

### 2.2 Trigger IDs

| Trigger ID | Event | Transition |
|------------|-------|------------|
| `start_df_event` | calendar OFF→ON | IDLE → Stage 1 |
| `df_event_started` | df_event_status OFF→ON | Stage 1 → Stage 2 |
| `take climate snapshot` | enable_climate_snapshot OFF→ON | Stage 2 → Stage 3 |
| `Enable DF conditions` | set_df_conditions OFF→ON | Stage 3 → Stage 4 |
| `df_event_ended` | calendar ON→OFF | DF Active → Restoration |
| `Occupant Overrides In App` | df_restriction_status ON→OFF | DF Active → Restoration |
| `Occupant Overrides On Device` | set_point_changed_on_device OFF→ON | DF Active → IDLE |
| `Occupant On Device Override - *` | thermostat attr changes | Triggers override detection |

### 2.3 Calendar Event Format

The automation expects calendar event descriptions in JSON format:

```json
{
  "T_delta": 3,
  "season": "cooling"
}
```

Where:
- `T_delta` (float): Temperature offset in °F
- `season` (string): Either `"cooling"` (raise setpoints) or `"heating"` (lower setpoints)

---

## 3. Critical Issues Identified

### 3.1 Issue Summary Table

| Issue | Severity | Location | Description |
|-------|----------|----------|-------------|
| **Global Variable Evaluation** | 🔴 Critical | Lines 131-156 | Variables evaluate on every trigger, including manual overrides |
| **Unsafe JSON Parsing** | 🔴 Critical | Lines 133-156 | `from_json` crashes if description is null/empty/malformed |
| **Snapshot Source Mismatch** | 🔴 Critical | Lines 203-210, 280-290 | Reads from `snapshot_thermostat` (climate entity) but values stored in `input_text` helpers |
| **No Self-Trigger Guard** | 🔴 Critical | Lines 108-119 triggers | Setting temperature fires override triggers immediately |
| **Arithmetic with None** | 🟠 High | Lines 203, 224 | `None + float` causes TypeError |
| **Parallel Mode Race Conditions** | 🟡 Medium | Line 325 | Simultaneous triggers can conflict |

### 3.2 Detailed Issue Analysis

#### 3.2.1 Global Variable Evaluation Problem

The original blueprint defines variables at the global `action:` level:

```yaml
action:
  - variables:
      t_delta: |-
        {% if is_state(df_calendar, 'on') %}
          {{ float((state_attr(df_calendar, 'description') | from_json)['T_delta'], 0) }}
        {% else %}
          0
        {% endif %}
```

**Problem:** These variables are evaluated for EVERY trigger, including when an occupant manually changes the thermostat. If the calendar description is null or invalid JSON at that moment, the automation crashes even though the variable isn't needed for override handling.

**Impact:** The error message "errored out in calendar entity reading attributes" occurs during unrelated manual thermostat adjustments.

#### 3.2.2 Snapshot Source Mismatch

The blueprint defines two thermostat references:

```yaml
variables:
  real_thermostat: !input climate_thermostat
  snapshot_thermostat: !input climate_thermostat_snapshot
```

During snapshot capture (Stage 3), values are written to `input_text` helpers:

```yaml
- action: input_text.set_value
  data:
    value: "{{ state_attr(real_thermostat, 'target_temp_high') }}"
  target:
    entity_id: !input thermostat_snapshot_target_temp_high
```

But during DF condition application (Stage 4), the code reads from `snapshot_thermostat` (a climate entity):

```yaml
- action: climate.set_temperature
  data: |
    {
      "target_temp_high": {{ state_attr(snapshot_thermostat, 'target_temp_high') + t_delta }}
    }
```

**Problem:** The snapshot values are stored in `input_text` entities, but the setpoint calculation reads from a different climate entity that was never populated with the snapshot data.

#### 3.2.3 Self-Triggering Override Detection

The automation monitors thermostat attribute changes to detect occupant overrides:

```yaml
- platform: state
  entity_id: !input climate_thermostat
  attribute: target_temp_high
  id: Occupant On Device Override - Target High Temp
```

**Problem:** When the automation calls `climate.set_temperature`, it changes these attributes, immediately firing the override detection trigger. The 5-second delay before setting `df_conditions_set` is insufficient because the trigger fires instantly when the attribute changes.

---

## 4. State Machine Model

### 4.1 Summary State Diagram

```
                    ┌─────────────────────────────────────┐
                    │              IDLE                   │
                    │        (All booleans OFF)           │
                    └─────────────────┬───────────────────┘
                                      │ calendar ON
                                      ▼
                    ┌─────────────────────────────────────┐
                    │         DF EVENT ACTIVE             │
                    │  ┌─────────────────────────────┐    │
                    │  │ Initialization               │    │
                    │  │ (start_df_event,            │    │
                    │  │  df_event_started)          │    │
                    │  └─────────────┬───────────────┘    │
                    │                ▼                    │
                    │  ┌─────────────────────────────┐    │
                    │  │ Snapshot Acquisition         │    │
                    │  │ (take climate snapshot)     │    │
                    │  └─────────────┬───────────────┘    │
                    │                ▼                    │
                    │  ┌─────────────────────────────┐    │
                    │  │ Setpoint Override           │    │
                    │  │ (Enable DF conditions)      │    │
                    │  └─────────────┬───────────────┘    │
                    │                ▼                    │
                    │  ┌─────────────────────────────┐    │
                    │  │ Steady State                │    │
                    │  │ (monitoring)                │    │
                    │  └─────────────────────────────┘    │
                    └───────┬───────────────────┬─────────┘
                            │                   │
              setpoint Δ    │                   │ calendar OFF
                            ▼                   ▼
                    ┌───────────────┐   ┌───────────────┐
                    │   OVERRIDE    │   │  RESTORATION  │
                    │   DETECTED    │   │               │
                    └───────┬───────┘   └───────┬───────┘
                            │                   │
                            └─────────┬─────────┘
                                      ▼
                                    IDLE
```

### 4.2 Stage Progression

| Stage | Trigger ID | Booleans ON After | Description |
|-------|------------|-------------------|-------------|
| IDLE | — | None | Awaiting calendar trigger |
| Stage 1 | `start_df_event` | `df_event_status` | Calendar turned ON |
| Stage 2 | `df_event_started` | +`df_restriction_status`, +`enable_climate_snapshot` | Restrictions enabled |
| Stage 3 | `take climate snapshot` | +`set_df_conditions` | Snapshot captured |
| Stage 4 | `Enable DF conditions` | +`df_conditions_set` | DF setpoints applied |
| Steady | — | All 5 ON | Monitoring for exit |
| Override | On-device change | +`set_point_changed_on_device` | User intervention |
| Exit | `df_event_ended` or override | All OFF | Return to IDLE |

### 4.3 Boolean State Vector by Stage

| Stage | 1 | 2 | 3 | 4 | 5 | 6 |
|:------|:-:|:-:|:-:|:-:|:-:|:-:|
| IDLE | ⚫ | ⚫ | ⚫ | ⚫ | ⚫ | ⚫ |
| Stage 1 | 🟢 | ⚫ | ⚫ | ⚫ | ⚫ | ⚫ |
| Stage 2 | 🟢 | 🟢 | 🟢 | ⚫ | ⚫ | ⚫ |
| Stage 3 | 🟢 | 🟢 | 🟢 | 🟢 | ⚫ | ⚫ |
| Stage 4+ | 🟢 | 🟢 | 🟢 | 🟢 | 🟢 | ⚫ |
| Override | ⚫ | ⚫ | ⚫ | ⚫ | ⚫ | ⚫ |

Legend:
1. `df_event_status`
2. `df_restriction_status`
3. `enable_climate_snapshot`
4. `set_df_conditions`
5. `df_conditions_set`
6. `set_point_changed_on_device`

---

## 5. Fault Analysis

### 5.1 Fault Classification

| Fault Code | Stage | Fault Type | Cause | Severity |
|------------|-------|------------|-------|----------|
| FAULT_01 | S1 | Sensor | Plant entity unavailable | 🟢 Benign |
| FAULT_02 | S1 | Mode | Plant in invalid mode (off/fan_only) | 🟢 Benign |
| FAULT_03 | S1 | Actuator | input_boolean service fails | 🟡 Minor |
| FAULT_04 | S2 | Actuator | Multiple boolean service fails | 🟡 Minor |
| FAULT_05 | S3 | Guard | df_event_status externally cleared | 🟡 Minor |
| FAULT_06 | S3 | Sensor | Plant attribute returns NULL | 🟠 Major |
| FAULT_07 | S3 | Memory | input_text service fails | 🟡 Minor |
| FAULT_08 | S4 | Reference | JSON parse error | 🟠 Major |
| FAULT_09 | S4 | Reference | Missing T_delta or season | 🟠 Major |
| FAULT_10 | S4 | Mode | Season ≠ HVAC mode | 🟠 Major |
| FAULT_11 | S4 | Guard | Guard boolean fails | 🔴 Critical |
| FAULT_12 | S4 | Plant | climate.set_temperature fails | 🟡 Minor |
| FAULT_13 | S4 | Range | Setpoint outside limits | 🟡 Minor |
| FAULT_14 | S4 | Guard | Guard stuck ON | 🔴 Critical |
| FAULT_15 | S6 | Guard | Guard stuck (override missed) | 🟠 Major |
| FAULT_16 | S6 | Detection | False positive | 🟠 Major |
| FAULT_17 | S7 | Actuator | Partial interlock release | 🟡 Minor |
| FAULT_18 | S8 | Guard | Condition check fails | 🟡 Minor |
| FAULT_19 | S8 | Memory | Snapshot corrupted | 🔴 Critical |
| FAULT_20 | S8 | Plant | Restore service fails | 🟡 Minor |
| FAULT_21 | S8 | Guard | Guard stuck ON | 🔴 Critical |
| FAULT_22 | S8 | Actuator | Partial flag reset | 🟡 Minor |

### 5.2 Silent Failure Chain

The most insidious fault pattern is **FAULT_06 → FAULT_19**:

```
Stage 3: Thermostat attribute returns None
    ↓
Writes "None" string to input_text (no error logged)
    ↓
Automation proceeds normally through S4, S5
    ↓
Stage 8: Restore attempts float("None")
    ↓
Either: crash OR sets thermostat to 0°F
```

This pattern is particularly dangerous because there's no immediate error—the failure only manifests hours later during restoration.

### 5.3 Stuck State Detection

Stuck states can be identified by checking boolean combinations:

| Boolean Combination | Stuck At | Likely Cause |
|---------------------|----------|--------------|
| `df_event_status=ON`, all others `OFF` | After S1 | FAULT_03, FAULT_04 |
| 3 booleans ON, `set_df_conditions=OFF` | After S3 | FAULT_06, FAULT_07 |
| 4 booleans ON, `df_conditions_set=OFF` | During S4 | FAULT_08, FAULT_09, FAULT_10 |
| All 5 ON for >4 hours after calendar OFF | After S5 | FAULT_18, FAULT_20, FAULT_22 |

---

## 6. Guard Signal Mechanism

### 6.1 The Problem Without Guard

When the automation calls `climate.set_temperature`, it changes the thermostat's `temperature`, `target_temp_high`, or `target_temp_low` attributes. These changes immediately fire the override detection triggers, causing the automation to incorrectly interpret its own commanded changes as occupant overrides.

```
Timeline (Without Guard):
─────────────────────────────────────────────────────────────►
t=0     Automation: climate.set_temperature(72)
t=200ms Thermostat: attribute changes to 72
t=200ms Trigger: "Occupant On Device Override" FIRES
t=201ms Result: FALSE POSITIVE - DF event cancelled
```

### 6.2 The Guard Solution

A guard flag (`df_applying_setpoint`) creates a temporal window during which override detection is inhibited:

```
Timeline (With Guard):
─────────────────────────────────────────────────────────────►
t=0       Guard: turn ON (df_applying_setpoint := ON)
t=500ms   Automation: climate.set_temperature(72)
t=700ms   Thermostat: attribute changes to 72
t=700ms   Trigger: FIRES
t=701ms   Condition: guard ON? → YES → IGNORE (correct!)
t=3500ms  Guard: turn OFF (df_applying_setpoint := OFF)
t=3501ms+ Any change now = real user override
```

### 6.3 Guard Timing Parameters

| Parameter | Value | Purpose |
|-----------|-------|---------|
| Guard ON → Command | 500ms | Ensure guard propagates before setpoint change |
| Command → Guard OFF (S4) | 3000ms | Allow plant to settle |
| Command → Guard OFF (S8) | 2000ms | Restore is simpler |
| Guard Timeout Max | 60000ms | Fault if guard ON longer than this |

### 6.4 Guard State Machine

```
            ┌───────────────┐    S4/S8 begins    ┌───────────────┐
            │               │ ──────────────────► │               │
            │  GUARD = OFF  │                     │  GUARD = ON   │
            │   (ENABLED)   │ ◄────────────────── │  (INHIBITED)  │
            │               │    S4/S8 ends       │               │
            └───────┬───────┘                     └───────┬───────┘
                    │                                     │
                    ▼                                     ▼
            ┌───────────────┐                     ┌───────────────┐
            │ Override Detect│                    │ Override Detect│
            │    ENABLED     │                    │    DISABLED    │
            │                │                    │                │
            │ Real override  │                    │ All changes    │
            │ → DETECTED     │                    │ → IGNORED      │
            └───────────────┘                     └───────────────┘
```

---

## 7. Monitoring Dashboard

### 7.1 Dashboard Overview

The monitoring dashboard provides real-time visibility into the DF automation state. It is designed to work with the **original automation** (without guard flag modifications).

### 7.2 Dashboard Sections

| Section | Purpose |
|---------|---------|
| **Event Timeline** | 24-hour history graph of all state flags |
| **DF System Status** | Current operating mode with stage description |
| **Stage Progress** | Checklist view of stage completion |
| **State Vector** | All 6 booleans with timestamps |
| **Calendar Event** | Parsed T_delta and season from calendar |
| **Computed Variables** | Real-time view of what automation would calculate |
| **Snapshot Memory** | Stored pre-DF thermostat values |
| **Snapshot vs Current** | Delta comparison between snapshot and live state |
| **Controls** | Reset button, override trigger, trace link |

### 7.3 Template Sensor for Operating Mode

To enable single-row history tracking of operating mode:

```yaml
template:
  - sensor:
      - name: "DF Operating Mode"
        unique_id: df_operating_mode
        icon: mdi:state-machine
        state: >-
          {% if is_state('input_boolean.df_event_status', 'off') %}
            IDLE
          {% elif is_state('input_boolean.set_point_changed_on_device', 'on') %}
            OVERRIDE
          {% elif is_state('input_boolean.df_conditions_set', 'on') %}
            DF_ACTIVE
          {% elif is_state('input_boolean.set_df_conditions', 'on') %}
            STAGE_4
          {% elif is_state('input_boolean.enable_climate_snapshot', 'on') %}
            STAGE_3
          {% elif is_state('input_boolean.df_restriction_status', 'on') %}
            STAGE_2
          {% else %}
            STAGE_1
          {% endif %}
```

### 7.4 Reset Script

```yaml
script:
  reset_df_state:
    alias: "DF Controller - Reset All Flags"
    sequence:
      - service: input_boolean.turn_off
        target:
          entity_id:
            - input_boolean.df_event_status
            - input_boolean.df_restriction_status
            - input_boolean.enable_climate_snapshot
            - input_boolean.set_df_conditions
            - input_boolean.df_conditions_set
            - input_boolean.set_point_changed_on_device
```

---

## 8. Deployment Task List

### 8.1 Pre-Deployment Tasks

| ID | Task | Est. Time |
|:---|:-----|:----------|
| DF-001 | Create entity mapping spreadsheet for all HA instances | 2h |
| DF-002 | Document naming convention for multi-thermostat deployments | 1h |
| DF-003 | Create base dashboard YAML template with placeholder variables | 2h |
| DF-004 | Create template sensor YAML for `sensor.df_operating_mode` | 30m |

### 8.2 Per-Instance Tasks (Single Thermostat)

| ID | Task | Est. Time | Dependencies |
|:---|:-----|:----------|:-------------|
| DF-010 | Inventory existing entities | 30m | DF-001 |
| DF-011 | Create input_boolean helpers (6 flags) | 15m | DF-010 |
| DF-012 | Create input_text helpers (7 snapshot fields) | 15m | DF-010 |
| DF-013 | Create template sensor (df_operating_mode) | 15m | DF-011 |
| DF-014 | Configure DF calendar entity | 30m | — |
| DF-015 | Deploy dashboard YAML with entity substitution | 30m | DF-011 to DF-014 |
| DF-016 | Deploy DF automation blueprint | 30m | DF-011, DF-012, DF-014 |
| DF-017 | Validate dashboard renders correctly | 15m | DF-015 |
| DF-018 | Test full DF event cycle | 1h | DF-016, DF-017 |

### 8.3 Per-Instance Tasks (Multi-Thermostat)

Additional tasks for instances with multiple thermostats:

| ID | Task | Est. Time | Dependencies |
|:---|:-----|:----------|:-------------|
| DF-020 | Inventory all thermostat entities | 30m | DF-001 |
| DF-021 | Define thermostat grouping strategy | 30m | DF-020 |
| DF-022 | Create input_boolean helpers per thermostat (6 × N) | 30m | DF-021 |
| DF-023 | Create input_text helpers per thermostat (7 × N) | 30m | DF-021 |
| DF-024 | Create template sensors per thermostat | 30m | DF-022 |
| DF-025 | Configure shared DF calendar entity | 30m | — |
| DF-026 | Deploy dashboard with tabbed thermostat views | 1h | DF-022 to DF-025 |
| DF-027 | Deploy DF automation blueprint per thermostat | 1h | DF-022, DF-023, DF-025 |
| DF-028 | Create aggregated status view | 1h | DF-026 |
| DF-029 | Validate all thermostat dashboards render | 30m | DF-026 |
| DF-030 | Test full DF event cycle per thermostat | 2h | DF-027, DF-029 |

### 8.4 Naming Convention

**Recommended Pattern:**
```
input_boolean.df_{home_id}_{zone}_{flag_name}
input_text.df_{home_id}_{zone}_snapshot_{field}
sensor.df_{home_id}_{zone}_operating_mode
automation.df_{home_id}_{zone}_full_cycle
```

**Example for Home 3, Kitchen Zone:**
```
input_boolean.df_home3_kitchen_event_status
input_text.df_home3_kitchen_snapshot_hvac_mode
sensor.df_home3_kitchen_operating_mode
automation.df_home3_kitchen_full_cycle
```

### 8.5 Acceptance Criteria

Each deployment must pass:

- [ ] All 6 input_boolean entities created and visible
- [ ] All 7 input_text snapshot helpers created
- [ ] Template sensor shows correct state
- [ ] Calendar entity configured with valid JSON description
- [ ] Dashboard loads without errors
- [ ] History graph shows all entities
- [ ] Stage Progress table updates in real-time
- [ ] Computed Variables parse calendar JSON correctly
- [ ] Reset All Flags button works
- [ ] Full DF cycle test passes: IDLE → STAGE 1-4 → DF_ACTIVE → IDLE

---

## 9. Appendices

### 9.1 Entity Mapping Template

| Placeholder | Your Entity |
|:------------|:------------|
| `calendar.df_weh` | `calendar._______` |
| `climate.thermostat` | `climate._______` |
| `input_boolean.df_event_status` | `input_boolean._______` |
| `input_boolean.df_restriction_status` | `input_boolean._______` |
| `input_boolean.enable_climate_snapshot` | `input_boolean._______` |
| `input_boolean.set_df_conditions` | `input_boolean._______` |
| `input_boolean.df_conditions_set` | `input_boolean._______` |
| `input_boolean.set_point_changed_on_device` | `input_boolean._______` |
| `input_text.thermostat_snapshot_hvac_mode` | `input_text._______` |
| `input_text.thermostat_snapshot_preset_mode` | `input_text._______` |
| `input_text.thermostat_snapshot_fan_mode` | `input_text._______` |
| `input_text.thermostat_snapshot_hvac_action` | `input_text._______` |
| `input_text.thermostat_snapshot_temperature` | `input_text._______` |
| `input_text.thermostat_snapshot_target_temp_high` | `input_text._______` |
| `input_text.thermostat_snapshot_target_temp_low` | `input_text._______` |
| `sensor.df_operating_mode` | `sensor._______` |

### 9.2 Control Systems Terminology Reference

| Term | DF Automation Equivalent |
|------|--------------------------|
| **Plant** | Thermostat / HVAC system |
| **Controller** | Home Assistant automation |
| **Reference Input r(t)** | Calendar event (T_delta, season) |
| **Control Output u(t)** | `climate.set_temperature` command |
| **Plant Output y(t)** | Thermostat attribute states |
| **Feedback** | Override detection (Stage 6) |
| **Interlock** | Boolean flags preventing unsafe transitions |
| **Guard** | `df_applying_setpoint` (inhibits feedback during command) |
| **Setpoint SP** | Target temperature |
| **Fault** | Error condition requiring intervention |

### 9.3 Related Artifacts

The following artifacts were created during this analysis:

1. **Summary DF State Diagram** - High-level state machine visualization
2. **DF Monitor Dashboard (No HACS)** - Lovelace YAML for monitoring
3. **DF Reset Script & Helpers** - Required entities configuration
4. **DF Dashboard Deployment Tasks** - Jira-style task list
5. **Guard Signal Timing Diagram** - Sequence diagram for guard mechanism
6. **Guard Signal Waveforms** - Timing analysis document

---

## Document History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | December 2024 | Initial comprehensive documentation |

---

*End of Document*
