# The MA 01 house DF test automation uses the following entities

## Entities

### Calendar
- Using the Google calendar events to manage the start and end time of the DF events. This calendar entity is generarted from [google calendar HA integration](https://www.home-assistant.io/integrations/google/). The integration allows read and write events for a calendar attached with the google acct the HA server is authorized to access. 

Follow the google calendar integration documentation to set up the integration.

#### For DF event specific directions: 
    - only eable the calendar entity you want to use for the DF event, disable the rest. 
    - the google calendar event should have a description in a JSON string format with keys - t_delta and season for the DF automation to set appropriate offsets. 


 - calendar.df_weh
    
#### Google caledar example DF event  
![alt text](image.png)
#### Google caledar example HA entity
![alt text](image-1.png)


### Climate
- climate.thermostat
- climate.thermostat_snapshot

as of 7/31: 
this is automaation/blueprint is set up to be for just one thermostat poer house. The snapshot climate entity uses the HACS integration [climate-template](https://github.com/jcwillox/hass-template-climate)


### Input Texts
- input_boolean.set_df_conditions
- input_text.thermostat_snapshot_fan_mode
- input_text.thermostat_snapshot_hvac_mode
- input_text.thermostat_snapshot_preset_mode
- input_text.thermostat_snapshot_hvac_action
- input_text.thermostat_snapshot_target_temp_low
- input_text.thermostat_snapshot_target_temp_high
- input_text.thermostat_snapshot_temperature

this are all the helper entities for setting the climate template entity for snapshots of pre DF climate conditions of the `real` thermostat in the house. 

TODO: Proposal - may be this can be converted into one input text with a dictionary that can be converted into a JSON object similar to the survey response entity. This may also be the way to scale it to multiple thermostat houses. 

### Input Booleans
- input_boolean.df_event_status
- input_boolean.df_restriction_status
- input_boolean.climate_snapshot
- input_boolean.set_df_conditions
- input_boolean.df_conditions_set
- input_boolean.set_point_changed_on_device

These Toggle helper entities are used as triggers for the automation that executes different stages of the DF event
1. input_boolean.df_event_status
    - this boolean turns on when the calendar entity turns on - trigger name: `start_df_event`, 
    - also the automation uses this boolean's state as a trigger when it sets to 'on' from 'off': termed `df_event_started`
1. input_boolean.climate_snapshot
1. input_boolean.df_restriction_status
    - these booleans turn on when automation is triggered by `df_event_started`
    - also, Climate snapshot changing from 'off' to 'on' is used as the next trigger: termed `take_climate_snapshot`
    - `take_climate_snapshot`:
        - updates the text helper entities defined above,
        - turns on input_boolean.set_df_conditions
1. input_boolean.set_df_conditions
    - used as trigger: termed `Enable DF conditions`
    - triggerred when the boolean turns on from off 
    - the DF conditions are set on the actual climate entity
    - waits for 5 seconds to add delay
    - turns on the next boolean `df_conditions_set`
1. input_boolean.df_conditions_set

1. input_boolean.set_point_changed_on_device 

1. input_boolean.thermostat_set_point_changed_on_device


### 