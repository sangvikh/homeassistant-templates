# homeassistant-templates
Useful templates for Home Assistant

# Energy price (nordpool integration)

## Is price now among the cheapest n hours today?

Used for controlling water heater etc.

````
{{ state_attr("sensor.nordpool", "current_price") <= ((state_attr("sensor.nordpool", "today")) | sort)[n-1] }}
````

## Estimated Hourly Energy Consumption

````
{{ states("sensor.energy_consumed_current_hour") | float * 3700 / (now().minute*60 + now().second + 100) }}
````
Added 100s to prevent div/0 and to decrease sensitivity the first minutes of the hour.


## Additional costs

Additional costs for calculating power tariff including fixed and variable cost

### Nettleie

Nettleie is 0.38 from 22-06, else 0.50.
````
{% if now().hour >= 22 or now().hour < 6 %}
    {{ 0.38 }}
{% else %}
    {{ 0.50 }}
{% endif %}
````

### Nettleie med strømstøtte

Strømstøtte 90% over 0.875 ink mva.
s = -(p-0.875)*0.9

````
{% if current_price > 0.875 %}
    {% if now().hour >= 22 or now().hour < 6 %}
        {{ -(current_price - 0.875)*0.9 + 0.35 }}
    {% else %}
        {{ -(current_price - 0.875)*0.9 + 0.47 }}
    {% endif %}
{% else %}
    {% if now().hour >= 22 or now().hour < 6 %}
        {{ 0.35 }}
    {% else %}
        {{ 0.47 }}
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
        {{ ((states('input_number.power_limit') | float * 0.9 -
        states('sensor.energy_consumed_current_hour') | float) * 1000 / 230) |
        round }}
      device_id: d0845e85e758774181b76398a3063cab
  - if:
      - condition: template
        value_template: >-
          {{states('sensor.estimated_hourly_energy_consumption') | float >
          states('input_number.power_limit') | float * 0.98}}
    then:
      - type: turn_on
        device_id: 3f0567e5411787bd406f6e045bfe6968
        entity_id: light.vvb_disable
        domain: light
    else: []
mode: single
````

