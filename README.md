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
    [*] --> Initial: Start
    Initial --> DF_Event_Started: calendar.nuhome_df_test "on"
    DF_Event_Started --> Snapshot_and_Restriction_Enabled: input_boolean.df_event_status "on"
    Snapshot_and_Restriction_Enabled --> Take_Climate_Snapshot: input_boolean.climate_snapshot "on"
    Take_Climate_Snapshot --> Set_DF_Conditions: input_boolean.set_df_conditions "on"
    
    Set_DF_Conditions --> Occupant_Override_In_App: input_boolean.df_restriction_status "off"
    Occupant_Override_In_App --> [*]: Reset Climate Conditions & DF booleans
    
    Set_DF_Conditions --> Occupant_Override_On_Device: input_boolean.smart_cities_set_point_changed_on_device "on"
    Occupant_Override_On_Device --> [*]: Turn off DF Booleans
    
    Set_DF_Conditions --> Detect_Set_Point_Changes: climate.office attribute change
    Detect_Set_Point_Changes --> [*]: Turn on ON device Override Boolean
    
    Set_DF_Conditions --> DF_Event_Ended: calendar.nuhome_df_test "off"
    DF_Event_Ended --> [*]: Reset DF Booleans & DF booleans
```
