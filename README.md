# HA-notes
thin are my notes for deploying, managing, improving the HA servers in 26+ homes over 2+ years 


# Conducting a calendar based psuedo demand response event in 25 homes

## Entities adn services needed: 
# Entities in the Automation

## Input Booleans
- `input_boolean.df_event_status`
- `input_boolean.climate_snapshot`
- `input_boolean.df_conditions_set`
- `input_boolean.df_restriction_status`
- `input_boolean.smart_cities_set_point_changed_on_device`

## Input Texts
- `input_text.smart_cities_snapshot_fan_mode`
- `input_text.smart_cities_snapshot_hvac_mode`
- `input_text.smart_cities_snapshot_preset_mode`
- `input_text.smart_cities_snapshot_hvac_action`
- `input_text.smart_cities_snapshot_target_temp_low`
- `input_text.smart_cities_snapshot_target_temp_high`
- `input_text.smart_cities_snapshot_temperature`

## Calendar
- `calendar.weh_mpnnvs5ppatx_gmail_com`

## Climate
- `climate.office`

## Services
- `climate.set_temperature`
- `input_text.set_value`
- `input_boolean.turn_on`
- `input_boolean.turn_off`
- `ecobee.resume_program`

# State Diagram

## Initial State
- **State**: Idle

## Event: DF Event Started
- **Trigger**: `df_event_started` (calendar event turned on)
- **Action**: 
  - Turn on `df_event_status`, `climate_snapshot`, and `df_restriction_status`

## State: DF_Event_Status_On
- **Action**: Check thermostat mode

## Check Thermostat Mode
- **Condition**: Thermostat is in `heat_cool` mode
  - **Action**: Set `target_temp_high` and `target_temp_low`
  - **Next State**: DF_Conditions_Set
- **Condition**: Thermostat is in `heat` or `cool` mode
  - **Action**: Set `temperature`
  - **Next State**: DF_Conditions_Set

## State: DF_Conditions_Set
- **Condition**: `df_restriction_status` turned on
  - **Action**: Register climate snapshot in helper entities
- **Condition**: `inApp_user_override` or `df_event_ended` and `df_event_status` is on
  - **Action**: 
    - Turn off `climate_snapshot` and `df_conditions_set`
    - If `preset_mode` is not `temp`, resume program
    - If `preset_mode` is `temp`, check thermostat mode:
      - **Condition**: Thermostat is in `heat_cool` mode
        - **Action**: Restore `target_temp_high` and `target_temp_low`
        - **Next State**: Idle
      - **Condition**: Thermostat is in `heat` or `cool` mode
        - **Action**: Restore `temperature`
        - **Next State**: Idle
- **Condition**: `onDevice_user_override` and `df_event_status` is on
  - **Action**: 
    - Turn off `climate_snapshot`, `df_event_status`, `df_conditions_set`, and `df_restriction_status`
    - Delay 30 seconds
    - Turn off `smart_cities_set_point_changed_on_device`
    - **Next State**: Idle

## Event: Setpoint Change
- **Trigger**: Changes in `climate.office` attributes (`temperature`, `target_temp_high`, `target_temp_low`)
- **Condition**: `df_conditions_set` is on
  - **Action**: Register `onDevice_set_point_change`
  - **Next State**: DF_Conditions_Set


# State changes involved / Event bus
```
+-------------------+
|       Idle        |
+-------------------+
        |
        | df_event_started
        v
+-------------------------+
| DF Event Started        |
+-------------------------+
        |
        | Register Snapshot
        v
+-------------------------+
| Register Snapshot       |
+-------------------------+
        |
        | df_event_status "on"
        v
+-------------------------+
| DF Event Status On      |
+-------------------------+
        |
        | Check Thermostat Mode
        |
        | heat_cool          | heat or cool
        v                    v
+---------------------+  +---------------------+
| Set Dual Mode       |  | Set Single Mode     |
+---------------------+  +---------------------+
        |                    |
        | df_conditions_set  | df_conditions_set
        v                    v
+-------------------------+
| DF Conditions Set       |
+-------------------------+
        |                  |                     |
        | df_event_ended   | inApp_user_override | onDevice_user_override
        v                  v                     v
+-------------------------+  +-----------------+  +-------------------------+
| Turn Off DF Conditions  |  | Turn Off DF     |  | Turn Off DF Conditions  |
| when Calendar Event Ends|  | Conditions      |  | when OnDevice Override  |
+-------------------------+  |                  |  +-------------------------+
                            +------------------+                             
```


```mermaid
stateDiagram-v2
    [*] --> Start_DF_Event: Calendar event for DF turns on

    state Start_DF_Event {
        direction LR
        Calendar_on --> Take_Climate_Snapshot
        Take_Climate_Snapshot --> Restrict_UI: Take snapshot of climate conditions prior to DF event
        Restrict_UI --> Set_DF_Conditions
    }

    Start_DF_Event --> DF_Conditions_Set: Start DF event when calendar event is live

    state DF_Conditions_Set {
        direction LR
        DF_Event_Started --> Climate_Snapshot_Enabled: Turns on snapshot, restriction booleans
        Climate_Snapshot_Enabled --> Capture_Climate_Conditions: Gets Thermostat set point(s) depending on the climate mode
        Capture_Climate_Conditions --> Enable_DF_Conditions: Turns boolean on to set DF conditions next
        Enable_DF_Conditions --> DF_Conditions_Applied: Sets DF conditions
    }

    state DF_Conditions_Applied {
        direction LR
        Dual_Mode_Heating_Cooling --> Apply_Offset_Both_Setpoints: Sets Dual mode HVAC conditions
        Apply_Offset_Both_Setpoints --> Conditions_Set
        Single_Mode_Heating_Cooling --> Apply_Appropriate_Setpoints: Confirms single mode and appropriate DF season
        Apply_Appropriate_Setpoints --> Conditions_Set
    }

    DF_Conditions_Set --> UI_Override: InApp Override detected - reset to prior Climate conditions and turn off DF booleans
    DF_Conditions_Set --> On_Device_Override: On Device Override detected - turn off DF booleans

    state UI_Override {
        direction LR
        UI_Restriction --> Resume_Previous_Program: Confirms if Pre-DF climate conditions were on a schedule, resumes ecobee programs
        Resume_Previous_Program --> Reset_Booleans: Turns off DF booleans
    }

    state On_Device_Override {
        direction LR
        Device_Change_Detected --> Reset_Booleans: Detect set point changes when DF event is on
    }

    Conditions_Set --> Reset_Booleans

    Reset_Booleans --> [*]
```


