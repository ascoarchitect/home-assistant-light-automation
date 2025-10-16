# Home Assistant Adaptive Lighting

An intelligent lighting system that automatically adjusts color temperature and brightness based on sun position, time of day, and ambient light levels.

## Features

- **Dynamic Color Temperature**: Smoothly transitions from warm (2700K) to cool (4000K) based on sun elevation using mired-based interpolation for perceptually uniform color transitions
- **Smart Brightness Control**: Adjusts brightness based on multiple conditions with priority-based logic
- **Sunrise/Sunset Awareness**: Special handling for dawn and dusk transitions
- **Ambient Light Detection**: Responds to low light conditions during the day
- **Night Mode**: Automatic dimming in evening hours

## Requirements

- Home Assistant
- Sun integration (built-in)
- A luminance/lux sensor (e.g., motion sensor with illuminance capability)

## Installation

### Step 1: Add Template Sensors

Add the following to your `configuration.yaml`:

```yaml
template:
  - sensor:
      - name: "Adaptive Light Temperature"
        unique_id: adaptive_light_temperature
        unit_of_measurement: "K"
        state: >
          {% set elevation = state_attr('sun.sun', 'elevation') | float %}
          {% set min_kelvin = 2700 %}
          {% set max_kelvin = 4000 %}
          
          {# When sun is below horizon, use warm temperature #}
          {% if elevation <= 0 %}
            {{ min_kelvin }}
          {# When sun is above horizon, interpolate in mired space for perceptually uniform transitions #}
          {% else %}
            {# Convert Kelvin to mireds (note: inverted - higher K = lower mired) #}
            {% set min_mired = 1000000 / max_kelvin %}
            {% set max_mired = 1000000 / min_kelvin %}
            
            {# Normalize elevation to 0-1 range (assuming max useful elevation is 60°) #}
            {% set max_elevation = 60 %}
            {% set normalized = (elevation / max_elevation) | float %}
            {% set normalized = 1.0 if normalized > 1.0 else normalized %}
            
            {# Use sine curve for smoother transition in mired space #}
            {% set mired_range = max_mired - min_mired %}
            {% set sine_value = sin(normalized * 1.5708) %} {# 1.5708 = π/2 #}
            {% set mired = min_mired + (sine_value * mired_range) %}
            
            {# Convert back to Kelvin #}
            {% set kelvin = 1000000 / mired %}
            {{ kelvin | round(0) }}
          {% endif %}
      
      - name: "Adaptive Light Brightness"
        unique_id: adaptive_light_brightness
        unit_of_measurement: "%"
        state: >
          {% set now_time = now() %}
          {% set sunrise = as_timestamp(state_attr('sun.sun', 'next_rising')) | timestamp_custom('%Y-%m-%d %H:%M:%S') | as_datetime %}
          {% set sunset = as_timestamp(state_attr('sun.sun', 'next_setting')) | timestamp_custom('%Y-%m-%d %H:%M:%S') | as_datetime %}
          {% set lux = states('sensor.your_lux_sensor') | float(100) %}
          {% set minutes_to_sunset = (as_timestamp(sunset) - as_timestamp(now_time)) / 60 %}
          {% set sunrise_45_min_before = as_timestamp(sunrise) - (45 * 60) %}
          {% set in_sunrise_window = as_timestamp(now_time) >= sunrise_45_min_before and as_timestamp(now_time) <= as_timestamp(sunrise) %}
          {% set in_sunset_window = minutes_to_sunset <= 45 and minutes_to_sunset > 0 %}
          {% set current_hour = now_time.hour %}
          {% set current_minute = now_time.minute %}
          {% set current_time_decimal = current_hour + (current_minute / 60.0) %}
          {% if in_sunrise_window %}
            {% set elapsed_minutes = (as_timestamp(now_time) - sunrise_45_min_before) / 60 %}
            {% set progress = elapsed_minutes / 45 %}
            {% set brightness = 10 + (progress * 40) %}
            {{ brightness | round(0) }}
          {% elif current_time_decimal >= 22 or (current_time_decimal < 6 and not in_sunrise_window) %}
            10
          {% elif current_time_decimal >= 21 and current_time_decimal < 22 %}
            {% set elapsed_hours = current_time_decimal - 21 %}
            {% set progress = elapsed_hours / 1.0 %}
            {% set brightness = 50 - (progress * 40) %}
            {{ brightness | round(0) }}
          {% elif in_sunset_window %}
            80
          {% elif lux < 15 %}
            50
          {% else %}
            0
          {% endif %}
```

**Important**: Replace `sensor.your_lux_sensor` with your actual luminance sensor entity ID.

### Step 2: Restart Home Assistant

After adding the configuration, restart Home Assistant to create the sensors.

### Step 3: Verify Sensors

Check that the following sensors are available:
- `sensor.adaptive_light_temperature`
- `sensor.adaptive_light_brightness`

## Configuration

### Color Temperature Behavior

| Sun Position | Color Temperature | Notes |
|-------------|-------------------|-------|
| Below horizon (≤0°) | 2700K | Warm white |
| 0° to 60° elevation | 2700K → 4000K | Mired-based interpolation with sine curve for perceptually uniform transitions |
| Above 60° elevation | 4000K | Cool daylight |

**Why Mired-Based Interpolation?** 

Equally-spaced mired values provide better perceptual uniformity than equally-spaced Kelvin values. This means color transitions feel more natural and consistent throughout the day. The calculation interpolates in mired space (reciprocal of Kelvin) and converts back to Kelvin for device compatibility.

### Brightness Behavior (Priority Order)

Priority is from highest to lowest:

1. **Pre-sunrise window** (45 min before sunrise → sunrise): 10% → 50% gradual increase
2. **Night mode** (10pm → 45 min before sunrise): 10% constant
3. **Evening fade** (9pm → 10pm): 50% → 10% gradual decrease
4. **Sunset window** (45 min before sunset): 80% constant
5. **Low ambient light** (when lux < 15): 50%
6. **Daytime** (default): 0% (lights off)

## Usage Examples

### Basic Automation with Motion Detection

```yaml
alias: Hallway Lights On with Motion
triggers:
  - type: motion
    device_id: YOUR_DEVICE_ID
    entity_id: YOUR_ENTITY_ID
    domain: binary_sensor
    trigger: device
actions:
  - action: light.turn_on
    data:
      color_temp_kelvin: "{{ states('sensor.adaptive_light_temperature') | int }}"
      brightness_pct: "{{ states('sensor.adaptive_light_brightness') | int }}"
      transition: 2
    target:
      entity_id: light.hallway_pendants
mode: single
```

### Manual Light Control Script

```yaml
script:
  set_adaptive_lighting:
    sequence:
      - service: light.turn_on
        target:
          entity_id: light.your_light
        data:
          color_temp_kelvin: "{{ states('sensor.adaptive_light_temperature') | int }}"
          brightness_pct: "{{ states('sensor.adaptive_light_brightness') | int }}"
          transition: 3
```

### Continuous Adaptive Lighting Automation

This automation keeps lights updated as the adaptive values change throughout the day:

```yaml
automation:
  - alias: "Continuous Adaptive Lighting"
    trigger:
      - platform: state
        entity_id: sensor.adaptive_light_brightness
      - platform: state
        entity_id: sensor.adaptive_light_temperature
    condition:
      - condition: state
        entity_id: light.your_light
        state: 'on'
    action:
      - service: light.turn_on
        target:
          entity_id: light.your_light
        data:
          color_temp_kelvin: "{{ states('sensor.adaptive_light_temperature') | int }}"
          brightness_pct: "{{ states('sensor.adaptive_light_brightness') | int }}"
          transition: 5
```

## Customization

### Adjusting Color Temperature Range

Modify these values in the `Adaptive Light Temperature` sensor:

```yaml
{% set min_kelvin = 2700 %}  # Minimum (warm)
{% set max_kelvin = 4000 %}  # Maximum (cool)
{% set max_elevation = 60 %} # Sun angle for max brightness
```

### Adjusting Brightness Thresholds

Modify these values in the `Adaptive Light Brightness` sensor:

- **Lux threshold**: `{% elif lux < 15 %}` (change 15 to your preferred value)
- **Evening start time**: `{% elif current_time_decimal >= 21 %}` (21 = 9pm)
- **Night start time**: `{% elif current_time_decimal >= 22 %}` (22 = 10pm)
- **Sunset window**: Change `45` to different number of minutes
- **Sunrise window**: Change `45` to different number of minutes

### Adjusting Brightness Levels

Current brightness levels:
- Night mode: `10%`
- Pre-sunrise end/Evening start: `50%`
- Sunset window: `80%`
- Low light: `50%`
- Daytime: `0%`

## Troubleshooting

### Sensors not appearing
- Verify the configuration syntax is correct
- Check Configuration → Logs for errors
- Ensure you've restarted Home Assistant

### Brightness always at 0%
- Check that your lux sensor entity ID is correct
- Verify the lux sensor is reporting values
- Ensure current time is not during daytime hours with good lighting

### Color temperature not changing
- Verify the sun integration is working (`sun.sun` entity exists)
- Check that your lights support color temperature control
- Ensure elevation attribute is available: `state_attr('sun.sun', 'elevation')`

### Lights not responding to automation
- Verify you're using the correct template syntax: `"{{ states('sensor.name') | int }}"`
- Check that the light entity supports both `brightness_pct` and `color_temp_kelvin`
- Review automation traces in Home Assistant for errors

## Advanced Features

### Alternative Color Temperature Calculation

If you prefer to output mireds directly (some lights accept `color_temp` in mireds), use this alternative sensor:

```yaml
- name: "Adaptive Light Temperature Mireds"
  unique_id: adaptive_light_temperature_mireds
  unit_of_measurement: "mired"
  state: >
    {% set elevation = state_attr('sun.sun', 'elevation') | float %}
    {% set min_kelvin = 2700 %}
    {% set max_kelvin = 4000 %}
    {% set min_mired = 1000000 / max_kelvin %}
    {% set max_mired = 1000000 / min_kelvin %}
    {% if elevation <= 0 %}
      {{ max_mired | round(0) }}
    {% else %}
      {% set max_elevation = 60 %}
      {% set normalized = (elevation / max_elevation) | float %}
      {% set normalized = 1.0 if normalized > 1.0 else normalized %}
      {% set sine_value = sin(normalized * 1.5708) %}
      {% set mired = min_mired + (sine_value * (max_mired - min_mired)) %}
      {{ mired | round(0) }}
    {% endif %}
```

Then use in automations with:
```yaml
color_temp: "{{ states('sensor.adaptive_light_temperature_mireds') | int }}"
```

## Contributing

Feel free to modify and adapt this configuration to your needs. Common improvements:
- Add different brightness profiles for different rooms
- Integrate with presence detection for automatic on/off
- Add manual override switches
- Create dashboard cards for easy control
- Adjust timing and thresholds to match your daily routine

## License

This configuration is provided as-is for use in Home Assistant installations.
