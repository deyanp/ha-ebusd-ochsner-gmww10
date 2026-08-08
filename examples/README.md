# Examples

Installation-specific pieces that sit on top of the ebusd configuration in
[`../config/`](../config/). None of them are required to read the bus — they are
here because working them out from scratch takes longer than adapting them.

| File | What it does |
|---|---|
| [`configuration.yaml`](configuration.yaml) | MQTT sensors for the values HA discovery skips (flow rates, energy, counters) and a writable operating-mode select |
| [`template_sensor.yaml`](template_sensor.yaml) | Folds the three status registers into one "what is it doing" sensor |
| [`automation_pv_surplus.yaml`](automation_pv_surplus.yaml) | Raises the DHW setpoint during PV surplus and puts it back afterwards |

Check the entity IDs before pasting: they follow the message names in `15.csv`,
so if you renamed anything there, they change with it. This lists yours:

```jinja
{% for s in states.sensor if 'ochsner' in s.entity_id %}{{ s.entity_id }} = {{ s.state }}
{% endfor %}
```

## A word on the writable ones

The DHW setpoint is safe to automate: it is a request, and the controller keeps
its own minimum off-time, defrost and pressure logic. Test any write once by
hand and confirm the value on the OTE panel before wiring it into an automation
— eBUS has no authentication, so a wrong register or a wrong scale factor is
written just as willingly as a right one.

The operating-mode select genuinely disables the heat generator. Worth having
for a deliberate block; not worth putting somewhere it can be hit by accident.
