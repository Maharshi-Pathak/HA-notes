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
stateDiagram
    [*] --> Idle
    
    Idle --> DF_Event_Started: df_event_started
    DF_Event_Started --> DF_Event_Status_On: Turn on df_event_status, climate_snapshot, df_restriction_status
    DF_Event_Status_On --> Check_Thermostat_Mode: Check thermostat mode
    Check_Thermostat_Mode --> Dual_Mode: Thermostat is heat_cool
    Dual_Mode --> Set_Temperature_Dual: Set target_temp_high and target_temp_low
    Check_Thermostat_Mode --> Single_Mode: Thermostat is heat or cool
    Single_Mode --> Set_Temperature_Single: Set temperature
    Set_Temperature_Dual --> DF_Conditions_Set: df_conditions_set "on"
    Set_Temperature_Single --> DF_Conditions_Set: df_conditions_set "on"

    DF_Conditions_Set --> Climate_Snapshot_Turned_On: df_restriction_status "on"
    Climate_Snapshot_Turned_On --> Register_Snapshot: Register climate snapshot
    Register_Snapshot --> DF_Conditions_Set

    DF_Event_Started --> InApp_User_Override: df_restriction_status "off"
    DF_Event_Started --> DF_Event_Ended: df_event_ended
    InApp_User_Override --> Handle_InApp_User_Override: df_event_status "on"
    Handle_InApp_User_Override --> Resume_Program: Resume program if preset_mode is not temp
    Handle_InApp_User_Override --> Restore_Temperatures: Restore temperatures if preset_mode is temp
    Restore_Temperatures --> Heat_Cool_Mode: Thermostat is heat_cool
    Heat_Cool_Mode --> Restore_Heat_Cool: Set target_temp_high and target_temp_low
    Restore_Temperatures --> Heat_Or_Cool_Mode: Thermostat is heat or cool
    Heat_Or_Cool_Mode --> Restore_Heat_Or_Cool: Set temperature
    Restore_Heat_Cool --> Idle
    Restore_Heat_Or_Cool --> Idle

    DF_Event_Started --> OnDevice_User_Override: smart_cities_set_point_changed_on_device "on"
    OnDevice_User_Override --> Handle_OnDevice_User_Override: df_event_status "on"
    Handle_OnDevice_User_Override --> Idle: Turn off all DF conditions

    DF_Conditions_Set --> Setpoint_Change: Changes in climate.office attributes
    Setpoint_Change --> Register_OnDevice_Set_Point_Change: Register set point change
    Register_OnDevice_Set_Point_Change --> DF_Conditions_Set
```
