# homeassistant-templates
Useful templates for Home Assistant

# Energy price (nordpool integration)

## Is price now among the cheapest n hours today?

Used for controlling water heater etc.

````
{{ state_attr("sensor.nordpool", "current_price") <= ((state_attr("sensor.nordpool", "today")) | sort)[n-1] }}
````

### Old VVB control
On the lowest 8 hours of the day.

````
{{state_attr("sensor.nordpool", "current_price") <= ((state_attr("sensor.nordpool", "today")) | sort)[states("input_number.vvb_hours_active") | int - 1]}}
````

### New VVB control
On the lowest 5 hours the next 12 hours.
If tomorrow is not ready, use the last 12 hours of the day after 12 o clock.

````
{% set n_hours = 5 %}
{% if not states('sensor.vvb_temperature') | is_number %}
    {{ true }}
{% elif states("sensor.vvb_temperature") | float < 30.0 %}
    {{ true }}
{% elif is_state_attr("sensor.nordpool", "tomorrow_valid", True) %}
    {{ state_attr("sensor.nordpool", "current_price") <= ((state_attr("sensor.nordpool", "today")  + state_attr("sensor.nordpool", "tomorrow"))[now().hour : now().hour + 12] | sort)[n_hours - 1] }}
{% else %}
    {% if now().hour < 13 %}
        {{ state_attr("sensor.nordpool", "current_price") <= ((state_attr("sensor.nordpool", "today"))[now().hour : now().hour + 12] | sort)[n_hours - 1] }}
    {% else %}
        {{ state_attr("sensor.nordpool", "current_price") <= ((state_attr("sensor.nordpool", "today"))[12 : 24] | sort)[n_hours - 1] }}
    {% endif %}
{% endif %}
````

## Estimated Hourly Energy Consumption

````
{{ states("sensor.energy_consumed_current_hour") | float * 3700 / (now().minute*60 + now().second + 100) }}
````
Added 100s to prevent div/0 and to decrease sensitivity the first minutes of the hour.

### As a sensor
Add this to configuration.yaml:
````
# Power estimate sensor
template:
  - sensor:
    - name: "Estimated Hourly Energy Consumption"
      unit_of_measurement: "kWh"
      unique_id: "energyestimate"
      device_class: "energy"
      state: '{{ states("sensor.energy_consumed_current_hour") | float * 3700 / (now().minute*60 + now().second + 100) }}'
````

## Additional costs

Additional costs for calculating power tariff including fixed and variable cost

### Nettleie

https://www.glitrenett.no/kunde/nettleie-og-priser/nettleiepriser-privatkunde

Nettleie is 0.3875 from 22-06, else 0.5075.
````
{% if now().hour >= 22 or now().hour < 6 %}
    {{ 0.41 }}
{% else %}
    {{ 0.53 }}
{% endif %}
````

### Nettleie med strømstøtte

Strømstøtte 90% over 0.9125 ink mva.
s = -(p-0.9125)*0.9

````
{% if current_price > 0.9125 %}
    {% if now().hour >= 22 or now().hour < 6 %}
        {{ -(current_price - 0.9125)*0.9 + states('input_number.energy_tariff_night') | float }}
    {% else %}
        {{ -(current_price - 0.9125)*0.9 + states('input_number.energy_tariff_day') | float }}
    {% endif %}
{% else %}
    {% if now().hour >= 22 or now().hour < 6 %}
        {{ states('input_number.energy_tariff_night') | float }}
    {% else %}
        {{ states('input_number.energy_tariff_day') | float }}
    {% endif %}
{% endif %}
````

## Events

### Use event data in template

Use event for trigger, then data can be accessed in a template.

````
{{ trigger.event.data.command == "off"}}
````

## Dynamic power limit
Adjust EV charger current to stay within specified power limit. Disable water heater as last resort

````
alias: Power Limiter
description: ""
mode: queued
max: 10
triggers:
  - entity_id:
      - sensor.estimated_hourly_energy_consumption
    trigger: state
conditions: []
actions:
  - data:
      current: >-
        {% set gain = 5 %} {{min(max(((states('input_number.power_limit') |
        float(0) * 1.0 - states('sensor.estimated_hourly_energy_consumption') |
        float(5)) * gain * 1000 / 230) | round(1), 0),16) }}
      device_id: d0845e85e758774181b76398a3063cab
    action: easee.set_charger_dynamic_limit
  - if:
      - condition: template
        value_template: >-
          {{states('sensor.estimated_hourly_energy_consumption') | float >
          states('input_number.power_limit') | float * 0.99}}
    then:
      - type: turn_off
        device_id: b7ce90c628132ce21939ed8e7b0053fb
        entity_id: 5b68f2d13c5fdc1052e3d5119de78717
        domain: switch
      - device_id: 9faf4e8cf8ae3b2448ef764ecdf5b496
        domain: climate
        entity_id: 7aa6d83c616568085f45cfd50850bae3
        type: set_hvac_mode
        hvac_mode: "off"
    else:
      - device_id: 9faf4e8cf8ae3b2448ef764ecdf5b496
        domain: climate
        entity_id: 7aa6d83c616568085f45cfd50850bae3
        type: set_hvac_mode
        hvac_mode: heat
````

