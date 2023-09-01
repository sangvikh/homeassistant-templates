# homeassistant-templates
Useful templates for Home Assistant

# Energy price (nordpool integration)

## Is price now among the cheapest n hours?

Used for controlling water heater etc.

````
{{ state_attr("sensor.nordpool", "current_price") <= ((state_attr("sensor.nordpool", "today")) | sort)[n-1] }}
````

## Estimated Hourly Energy Consumption

````
{{ states("sensor.energy_consumed_current_hour") | float * 3660 / (now().minute*60 + now().second + 60) }}
````
Added 60s to prevent div/0 and to fade estimate in slowly. Sensor update rate is about 15s.


## Additional costs

Additional costs for calculating power tariff including fixed and variable cost

### Nettleie

Nettleie is 0.35 from 22-06, else 0.47.
````
{% if now().hour >= 22 or now().hour < 6 %}
    {{ 0.35 }}
{% else %}
    {{ 0.47 }}
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
