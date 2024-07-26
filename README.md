# HA-notes
thin are my notes for deploying, managing, improving the HA servers in 26+ homes over 2+ years 


# Conducting a calendar based psuedo demand response event in 25 homes

## Entities adn services needed: 
### Input Booleans:
- `input_boolean.df_event_status`
- `input_boolean.df_restriction_status`
- `input_boolean.df_conditions_set`
- `input_boolean.climate_snapshot`
- `input_boolean.smart_cities_set_point_changed_on_device`

### Calendar:
- `calendar.weh_mpnnvs5ppatx_gmail_com`

### Climate Entities:
- `climate.office`
- `climate.smart_cities_snapshot_2`

### Input Text:
- `input_text.smart_cities_snapshot_preset_mode`
- `input_text.smart_cities_snapshot_fan_mode`
- `input_text.smart_cities_snapshot_hvac_mode`
- `input_text.smart_cities_snapshot_hvac_action`
- `input_text.smart_cities_snapshot_target_temp_low`
- `input_text.smart_cities_snapshot_target_temp_high`
- `input_text.smart_cities_snapshot_temperature`

### Services:
- `input_boolean.turn_on`
- `input_boolean.turn_off`
- `climate.set_temperature`
- `ecobee.resume_program`
- `input_text.set_value`

### Variables:
- `t_delta`
- `season`
- `df_delta_multiplier`


# State changes involved / Event bus
```
+-------------------+
|       Off         |
+-------------------+
        |                       | 
        | df_event_status "on"  | df_event_status "off"
        v                       v
+-------------------+       +-------------------+
| DF Event Started  | <-->  | DF Event Turned Off |
+-------------------+       +-------------------+
        |
        | climate_snapshot "on"
        v
+---------------------------+
| Climate Snapshot Turned On |
+---------------------------+
        |
        | climate_snapshot "off"
        v
+----------------------------+
| Climate Snapshot Turned Off |
+----------------------------+
        |
        | df_conditions_set "on"
        v
+----------------------------+
| DF Event Conditions Set     |
+----------------------------+
        |
        | changes in climate.office attributes and df_conditions_set "on"
        v
+----------------------------+
| Setpoint Change             |
+----------------------------+
        |
        | df_restriction_status "off"
        v
+-------------------+
| InApp User Override |
+-------------------+
        |
        | off
        v
+-------------------+
|       Off         |
+-------------------+
```

```mermaid
stateDiagram
        [*] --> Off
        
        Off --> DF_Event_Started: df_event_status "on"
        Off --> DF_Event_Turned_Off: df_event_status "off"
        Off --> DF_Event_Conditions_Set: df_conditions_set "on"
        
        DF_Event_Started --> Climate_Snapshot_Turned_On: climate_snapshot "on"
        Climate_Snapshot_Turned_On --> Climate_Snapshot_Turned_Off: climate_snapshot "off"
        
        DF_Event_Conditions_Set --> Setpoint_Change: changes in climate.office attributes and df_conditions_set "on"
        
        InApp_User_Override --> Off: df_restriction_status "off"
        
        DF_Event_Turned_Off --> Off
        
        DF_Event_Started --> InApp_User_Override: df_restriction_status "off"
        
        state Off {
        [*] --> Off
        }
        
        state DF_Event_Started {
        [*] --> DF_Event_Started
        }
        
        state Climate_Snapshot_Turned_On {
        [*] --> Climate_Snapshot_Turned_On
        }
        
        state Climate_Snapshot_Turned_Off {
        [*] --> Climate_Snapshot_Turned_Off
        }
        
        state DF_Event_Conditions_Set {
        [*] --> DF_Event_Conditions_Set
        }
        
        state Setpoint_Change {
        [*] --> Setpoint_Change
        }
        
        state InApp_User_Override {
        [*] --> InApp_User_Override
        }
        
        state DF_Event_Turned_Off {
        [*] --> DF_Event_Turned_Off
        }
```
