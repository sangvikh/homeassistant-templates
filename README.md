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
On the lowest 4 hours the next 12 hours.
If tomorrow is not ready, use the last 12 hours of the day after 12 o clock.

````
{% if is_state_attr("sensor.nordpool", "tomorrow_valid", True) %}
    {{ state_attr("sensor.nordpool", "current_price") <= ((state_attr("sensor.nordpool", "today")  + state_attr("sensor.nordpool", "tomorrow"))[now().hour : now().hour + 12] | sort)[4 - 1] }}
{% else %}
    {% if now().hour <= 12 %}
        {{ state_attr("sensor.nordpool", "current_price") <= ((state_attr("sensor.nordpool", "today"))[now().hour : now().hour + 12] | sort)[4 - 1] }}
    {% else %}
        {{ state_attr("sensor.nordpool", "current_price") <= ((state_attr("sensor.nordpool", "today"))[12:24] | sort)[4 - 1] }}
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

Nettleie is 0.3009 from 22-06, else 0.4209.
````
{% if now().hour >= 22 or now().hour < 6 %}
    {{ 0.3009 }}
{% else %}
    {{ 0.4209 }}
{% endif %}
````

### Nettleie med strømstøtte

Strømstøtte 90% over 0.9125 ink mva.
s = -(p-0.9125)*0.9

````
{% if current_price > 0.9125 %}
    {% if now().hour >= 22 or now().hour < 6 %}
        {{ -(current_price - 0.9125)*0.9 + 0.3009 }}
    {% else %}
        {{ -(current_price - 0.9125)*0.9 + 0.4209 }}
    {% endif %}
{% else %}
    {% if now().hour >= 22 or now().hour < 6 %}
        {{ 0.3009 }}
    {% else %}
        {{ 0.4209 }}
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
trigger:
  - platform: state
    entity_id:
      - sensor.estimated_hourly_energy_consumption
condition: []
action:
  - service: easee.set_charger_dynamic_limit
    data:
      current: >-
        {{min(max(((states('input_number.power_limit') | float * 1.0 -
        states('sensor.estimated_hourly_energy_consumption') | float) * 5 * 1000
        / 230) | round(0), 0),16) }}
      device_id: d0845e85e758774181b76398a3063cab
  - if:
      - condition: template
        value_template: >-
          {{states('sensor.estimated_hourly_energy_consumption') | float >
          states('input_number.power_limit') | float * 0.99}}
    then:
      - type: turn_on
        device_id: 3f0567e5411787bd406f6e045bfe6968
        entity_id: light.vvb_disable
        domain: light
    else: []
mode: single
````

