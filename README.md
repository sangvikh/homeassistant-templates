# homeassistant-templates
Useful templates for Home Assistant

# Energy price (nordpool integration)

## Is price now among the cheapest n hours?

````
{{ state_attr("sensor.nordpool", "current_price") < ((state_attr("sensor.nordpool", "today")) | sort)[n] }}
````

## Estimate end of hour power consumption

energy_consumed_current_hour is a utility meter of a riemann sum of the power used.

````
{{ states("sensor.energy_consumed_current_hour") | float * 60 / now().minute }}
````

## Additional costs

Additional costs for calculating power tariff including fixed and variable cost

### Nettleie

Nettleie is 0.35 from 22-06, else 0.47.
````
{% if now().hour >= 22 or now().hour < 6 %}
    {{ 0.35 | float }}
{% else %}
    {{ 0.47|float }}
{% endif %}
````

### Nettleie med strømstøtte

Strømstøtte 90% over 0.875 ink mva.
s = p - (p-0.875)*0.9 = 0.1*p + 0.7875

````
{% if state_attr("sensor.nordpool", "current_price") > 0.875 %}
    {% if now().hour >= 22 or now().hour < 6 %}
        {{ state_attr("sensor.nordpool", "current_price")*0.1 + 0.7875 + 0.35 | float }}
    {% else %}
        {{ state_attr("sensor.nordpool", "current_price")*0.1 + 0.7875 + 0.47|float }}
    {% endif %}
{% else %}
    {% if now().hour >= 22 or now().hour < 6 %}
        {{ 0.35 | float }}
    {% else %}
        {{ 0.47|float }}
    {% endif %}
{% endif %}
````
