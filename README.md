i wanted to have a little card showing where i currently am on our home assistant dash.
i used this as inspiration/base: https://github.com/CHirch/Traewelling-Home-Assistant

most of the logic in here is written by claude.

its currently using the AccessToken instead of OAuth, which, isn't pretty, but hey it works :p
you don't need the auth header at all if you have a public profile & only have public check-ins.
get the AccessToken [here](https://traewelling.de/settings/applications)

adapt this to your liking :D
(and make sure to adapt the sensor names in the progress sensor & the card, cuz some of those are gonna be different on your end)

here's my home assistant `configuration.yaml`

```yaml
# Loads default set of integrations. Do not remove.
default_config:

# Load frontend themes from the themes folder
frontend:
  themes: !include_dir_merge_named themes

automation: !include automations.yaml
script: !include scripts.yaml
scene: !include scenes.yaml

zha:
  custom_quirks_path: /config/custom_zha_quirks/

template:
  - sensor:
      - name: "Traewelling Progress"
        unique_id: traewelling_progress
        unit_of_measurement: "%"
        state_class: measurement
        state: >
          {% set dep = as_datetime(states('sensor.traewelling_origin_time_2')) %}
          {% set arr = as_datetime(states('sensor.traewelling_arrival_realtime')) %}
          {% set total = (arr - dep).total_seconds() %}
          {% set done = (now() - dep).total_seconds() %}
          {% if total > 0 %}
            {{ [ [ (done / total * 100) | round(0), 0 ] | max, 100 ] | min }}
          {% else %}
            0
          {% endif %}
        availability: >
          {{ has_value('sensor.traewelling_origin_time_2')
             and has_value('sensor.traewelling_arrival_realtime') }}
  - binary_sensor:
    - name: "Traewelling Check-In aktiv"
      unique_id: traewelling_check_in_active
      state: >
        {% set dep = as_datetime(states('sensor.traewelling_origin_time_2')) - timedelta(minutes=30) %}
        {% set arr = as_datetime(states('sensor.traewelling_arrival_realtime')) + timedelta(minutes=10) %}
        {{ dep <= now() <= arr }}
      availability: >
        {{ has_value('sensor.traewelling_origin_time_2')
           and has_value('sensor.traewelling_arrival_realtime') }}

# Traewelling REST sensors
rest:
  - method: GET
    resource: https://traewelling.de/api/v1/user/lucy
    headers:
      accept: application/json
      Authorization: "Bearer <AccessToken>"
    scan_interval: 900
    sensor:
      - name: Traewelling Total Distance
        value_template: "{{ (value_json.data.totalDistance / 1000) | round(1) }}"
        state_class: total_increasing
        unit_of_measurement: "km"
        icon: mdi:train
        unique_id: traewelling_distance
      - name: Traewelling Total Duration
        value_template: "{{ value_json.data.totalDuration }}"
        state_class: total_increasing
        unit_of_measurement: "min"
        icon: mdi:train
        unique_id: "traewelling_duration"
      - name: Traewelling Points
        value_template: "{{ value_json.data.points }}"
        icon: mdi:train
        state_class: total
        unique_id: traewelling_points

  - method: GET
    resource: https://traewelling.de/api/v1/user/lucy/statuses?page=1
    headers:
      accept: application/json
      Authorization: "Bearer <AccessToken>"
    scan_interval: 120
    sensor:
      - name: Traewelling Check-In-ID
        value_template: "{{ value_json.data[0].id }}"
        icon: mdi:train
        unique_id: traewelling_check_in_id
      - name: Traewelling Destination Name
        value_template: "{{ value_json.data[0].checkin.destination.name }}"
        icon: mdi:train
        unique_id: traewelling_destination_name
      - name: Traewelling Origin Name
        value_template: "{{ value_json.data[0].checkin.origin.name }}"
        icon: mdi:train
        unique_id: traewelling_origin_name
      - name: Traewelling Arrival PlannedTime
        value_template: "{{ value_json.data[0].checkin.destination.arrivalPlanned }}"
        icon: mdi:bus-clock
        unique_id: traewelling_arrival_plannedtime
      - name: Traewelling Origin PlannedTime
        value_template: "{{ value_json.data[0].checkin.origin.departurePlanned }}"
        icon: mdi:bus-clock
        unique_id: traewelling_origin_plannedtime
      - name: Traewelling Line
        value_template: "{{ value_json.data[0].checkin.lineName }}"
        icon: mdi:train
        unique_id: traewelling_line
      - name: Traewelling Arrival RealTime
        value_template: >
          {% set c = value_json.data[0].checkin %}
          {{ c.manualArrival or c.destination.arrivalReal or 'unknown' }}
        icon: mdi:bus-clock
        unique_id: traewelling_arrival_realtime
      - name: Traewelling Origin RealTime
        value_template: >
          {% set c = value_json.data[0].checkin %}
          {{ c.manualDeparture or c.origin.departureReal or 'unknown' }}
        icon: mdi:bus-clock
        unique_id: traewelling_origin_realtime
      - name: Traewelling Departure Delay
        value_template: >
          {% set c = value_json.data[0].checkin %}
          {% set real = c.manualDeparture or c.origin.departureReal %}
          {% if real and c.origin.departurePlanned %}
            {{ ((as_datetime(real) - as_datetime(c.origin.departurePlanned)).total_seconds() / 60) | round(0) | int }}
          {% else %}
            0
          {% endif %}
        unit_of_measurement: "min"
        state_class: measurement
        icon: mdi:clock-alert-outline
        unique_id: traewelling_departure_delay
      - name: Traewelling Arrival Delay
        value_template: >
          {% set c = value_json.data[0].checkin %}
          {% set real = c.manualArrival or c.destination.arrivalReal %}
          {% if real and c.destination.arrivalPlanned %}
            {{ ((as_datetime(real) - as_datetime(c.destination.arrivalPlanned)).total_seconds() / 60) | round(0) | int }}
          {% else %}
            0
          {% endif %}
        unit_of_measurement: "min"
        state_class: measurement
        icon: mdi:clock-alert-outline
        unique_id: traewelling_arrival_delay
```

and here's the custom home assistant card (wip):
```yaml
type: conditional
conditions:
  - condition: state
    entity: binary_sensor.traewelling_check_in_aktiv
    state: 'on'
card:
  type: markdown
  title: Where is Lu going?
  content: >
    {% set dep_plan  = 'sensor.traewelling_origin_time' %}
    {% set dep_real  = 'sensor.traewelling_origin_time_2' %}
    {% set dep_delay = 'sensor.traewelling_departure_delay' %}
    {% set arr_plan  = 'sensor.traewelling_arrival_plannedtime' %}
    {% set arr_real  = 'sensor.traewelling_arrival_realtime' %}
    {% set arr_delay = 'sensor.traewelling_arrival_delay' %}

    {% macro row(name, plan, real, delay) -%}

    {%- set pt = as_timestamp(states(plan), none) -%}
    {%- set rt = as_timestamp(states(real), none) -%}
    {%- set ps = pt | timestamp_custom('%H:%M') if pt else none -%}
    {%- set rs = rt | timestamp_custom('%H:%M') if rt else none -%}
    {%- set v = states(delay) | int(0) -%}

    {%- if ps and rs and ps != rs -%}
      {%- set txt = '~~' ~ ps ~ '~~ **' ~ rs ~ '** (' ~ ('+' if v > 0 else '') ~ v ~ ' min)' -%}
    {%- elif ps and rs -%}
      {%- set txt = ps ~ ' (on time)' -%}
    {%- else -%}
      {%- set txt = rs or ps or '–' -%}
    {%- endif -%}

    {{ name ~ ' · ' ~ txt }}

    {%- endmacro %}


    {% set p = states('sensor.traewelling_progress') | int(0) %}
    {% set full = [(p / 5) | int, 20] | min %}
    {% set bar = ('⣿' * 20) if p >= 100 else (('⣿' * full) ~ '⣦' ~ ('⣀' * (19 - full))) %}


    ### {{ states('sensor.traewelling_line') }}
    
    {{ row(states('sensor.traewelling_origin_name'), dep_plan, dep_real,
    dep_delay) }}

    ↳ {{ row(states('sensor.traewelling_destination_name'), arr_plan, arr_real,
    arr_delay) }}


    `{{ bar }}` {{ p }} %
```

