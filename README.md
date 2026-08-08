# Ochsner GMWW 10 plus (OTE plus) — ebusd configuration for Home Assistant

Working eBUS message definitions and a step-by-step setup guide for an
**Ochsner GMWW 10 plus** brine/water heat pump with an **OTE plus** controller,
read and written from Home Assistant over MQTT.

| | |
|---|---|
| Heat pump | Ochsner GMWW 10 plus, commissioned 2013 |
| Controller | OTE plus — software **5.02**, hardware **3.03** |
| eBUS identity | `MF=TEM ID=22420 SW=0502 HW=0102`, slave address **15** |
| Adapter | eBUS Adapter Shield v5 (C6), USB, enhanced protocol |
| Stack | ebusd 26.1 → MQTT → Home Assistant |

Roughly 40 values are read correctly and 12 registers are writable, including
the hot-water setpoint and the heat-generator enable — enough to drive the heat
pump from a PV-surplus automation without touching the manufacturer's control
logic.

### What the whole job looks like

1. Buy the adapter and a length of twisted-pair cable — [shopping list](#shopping-list)
2. Configure the adapter for USB + enhanced protocol — [step 1](#1-configure-the-adapter)
3. Tap the bus at the OTE control panel — [hardware](#hardware)
4. Install the ebusd add-on and point it at the adapter — [steps 2–3](#2-install-the-ebusd-add-on)
5. Drop these CSV files in and restart — [step 4](#4-install-these-definitions)
6. Check the values against the OTE display — [verifying](#verifying-and-debugging)
7. Build a card — [dashboard](#a-dashboard-card)

Budget an evening. Most of it is waiting for poll cycles.

## Why this repository exists

The upstream configuration
([wiedwo/ebusd-guide-Ochsner-GMWW-with-Home-Assistant](https://github.com/wiedwo/ebusd-guide-Ochsner-GMWW-with-Home-Assistant),
itself derived from [Lorilatschki](https://github.com/Lorilatschki/ebusd-guide)
and [cybersmart-eu](https://github.com/cybersmart-eu/ebusd-ochsner)) targets a
**GMWW 11 plus with OTE3, software 6.05**. On software **5.02** it loads, but a
number of values come out wrong or not at all.

Four classes of problem had to be fixed. All of them are documented below, in
case you land here with yet another firmware revision.

### 1. Message IDs are shifted by one

On this firmware, each message in the counter block returns the parameter with
the **next higher** number than the upstream name suggests. The `group` field in
every reply gives the truth — compare it against the parameter number shown on
the OTE display.

| eBUS ID | upstream name | actually returns |
|---|---|---|
| `7d860002` | `Waermepumpe.Betriebsstunden.02.081` | **02-080** cycle count |
| `7d870002` | `Volumenstrom.Waermenutzung.21.002` | **02-081** operating hours |
| `7d880002` | `Volumenstrom.Waermequelle.21.090` | **21-002** flow rate, heating |
| `7d890002` | `Heizenergie.kWh.23.001` | **21-090** flow rate, source |
| `7d8a0002` | `Heizenergie.MWh.23.010` | **23-001** heat energy kWh |
| `7d8b0002` | `Warmwasserenergie.kWh.23.066` | **23-010** heat energy MWh |
| `7d8c0002` | `Warmwasserenergie.MWh.23.013` | **23-006** DHW energy kWh |
| `7d8d0002` | *(not present upstream)* | **23-013** DHW energy MWh |

The same shift affects two temperatures:

| eBUS ID | upstream name | actually returns |
|---|---|---|
| `77830008` | `Aktueller.Raumsollwert.01.001` | **00-001** room temperature |
| `77840008` | `Istwert.Heizkreis.Vorlauftemp.00.002` | **01-001** room setpoint |
| `77850008` | `Sollwert.Heizkreis.Vorlauftemp.01.002` | **00-002** heating flow temperature |

`15.csv` here renames the counter block and `77830008`; the other two are left
under their upstream names so the diff stays small — mind the labels when you
build dashboards.

### 2. Counters need `HCD:4`, not `HCD:3`

Cycle counts and operating hours carry a fourth BCD byte on this firmware.
Upstream's `cycles` (`HCD`) and `hours` (`HCD:3`) truncate it: 19673 cycles were
reported as 960000, and 24896 operating hours as 196.

`_templates.csv` adds `cycles4` (`HCD:4`) and `hours4` (`HCD:4`, divider 100).

### 3. Two field templates are missing upstream

`modehotwater` and `wpbetrieb` are referenced by the message definitions but not
defined, which aborts the whole config load with
`ERR: element not found, field type MODEHOTWATER in field 1`. Both are added
here.

### 4. Write rows have the field in the wrong column

Every `w,ochsner,...` row upstream places its field type in column 16, where the
matching read row has it in column 11. ebusd accepts the row without complaint
but never registers a writable message — `ebusctl find -w` comes back empty.
All 12 write rows are corrected here.

---

## Shopping list

Everything needed, start to finish. Prices are what this build cost in Austria
in 2026; treat them as a rough guide.

| Item | Why | Approx. |
|---|---|---|
| **eBUS Adapter Shield v5** (C6, with enclosure) | The bus interface. Order from [adapter.ebusd.eu](https://adapter.ebusd.eu/) or a reseller such as BerryBase. The enclosure is worth it — the board sits in a technical room. | €50–60 |
| **USB-A to USB-C cable**, data-capable | Adapter to host. Charge-only cables will not enumerate. | €5 |
| **Cat-6 S/FTP installation cable**, ~15 m | Bus wiring. J-Y(St)Y 2x2x0.8 is the manufacturer's spec; Cat-6 S/FTP is equivalent or better and easier to buy. Solid core, AWG23. | €15 |
| **WAGO 221-413**, 2 | For bench testing at the control panel without disturbing existing wiring. Optional but strongly recommended before you commit to a permanent install. | €5 |

**Not needed:** the Ochsner Modbus gateway (ZIF180) — several hundred euros,
requires an Ochsner technician to enable it, and on some installations it takes
over the controller and disables the OTE panel. eBUS gives you the same data
passively for a tenth of the price.

**Already assumed present:** a Home Assistant instance, the Mosquitto broker
add-on, and the Advanced SSH & Web Terminal add-on. Studio Code Server or Samba
makes copying the config files easier but is not essential.


---

## Hardware

The eBUS is accessible at the **OTE control panel**, which lifts out of its
plastic cover without tools. On the back of the MB 6401 board:

- **red 4-pin connector**, silkscreened `eBus + -` — pins 41 and 42; measured
  about 20 V DC against ground
- **white screw terminal**, 2 wires — the same bus, in parallel
- the **right-hand free pin** of the red connector is continuous with screw 1;
  the left-hand free pin is *not* on the bus (about 3 V) — don't use it

For a bench test, mini-grabber test clips on the right-hand free pin and on
screw 2 tap the bus without disturbing any existing wiring. For permanent
installation, terminals 41/42 in the switch cabinet are the better place.

**Cable:** Ochsner specifies J-Y(St)Y 2x2x0.8. Cat-6 S/FTP works just as well
and is usually easier to get — use **both wires of one twisted pair**, one per
terminal. Never one wire each from two different pairs: the twist is what
rejects interference, and it is the main protection you have when the cable runs
alongside the heat pump's supply line.

**Shield:** bond to PE **at one end only**, in the heat pump's switch cabinet.
Cut it back and insulate it at the adapter end. Bonding both ends creates a loop
between two earth points that carries current and injects the very noise the
shield is there to keep out. If you are tapping at the control panel, where
there is no PE, leave the shield open at both ends.

**Adapter:** an [eBUS Adapter Shield v5](https://adapter.ebusd.eu/) — this build
uses the C6 (ESP32-C6) over USB. eBUS polarity is irrelevant: the adapter has a
bridge rectifier on its input.

---

## Setup

### 1. Configure the adapter

Ships pre-flashed. Power it up, connect to the open Wi-Fi access point `EBUS`,
open `http://192.168.4.1`, then under **Configuration → eBUS**:

- **Protocol:** Enhanced protocol
- **Connection:** USB serial

Save, reboot. The page shows the ebusd device string to use, e.g.
`ens:/dev/ttyACM0`.

Hold the button while powering up to force the access point back if you ever
lock yourself out. Update the firmware *after* you have data, not before —
there are reports of the eBUS signal disappearing after an update, and you want
a known-good baseline to compare against.

### 2. Install the ebusd add-on

Add the repository <https://github.com/LukasGrebe/ha-addons> in **Settings →
Add-ons → Add-on Store → Repositories**, then install **eBUSd**. Mosquitto must
already be running; the add-on picks up its credentials automatically.

### 3. Point it at the adapter

Leave the USB picker empty and put the full device string in the
**network / enhanced-protocol adapter** field — the plain USB field does not
carry the `enh:` prefix:

```
enh:/dev/serial/by-id/usb-Espressif_USB_JTAG_serial_debug_unit_XX:XX:XX:XX:XX:XX-if00
```

Under **additional ebusd options**, add one flag per entry:

```
--configpath=/config/ebusd
--httpport=8889
```

`--httpport` is not required for normal operation, but the JSON API it exposes
is how you inspect raw frames when a value looks wrong — see *Decoding a wrong
value* below.

Do **not** set `--pollinterval`. ebusd polls one message per interval, so a
30-second interval means a 22-minute round trip through 44 messages, and
everything reads as unknown for a long time after a restart. The 5-second
default is right.

If the UI drops flags when you save, write them through the Supervisor API
instead:

```bash
curl -s -X POST -H "Authorization: Bearer $SUPERVISOR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"options":{"seed_mqtt_cfg":true,"commandline_options":["--mqttjson","--configpath=/config/ebusd","--httpport=8889"],"network_device":"enh:/dev/serial/by-id/usb-Espressif_..."}}' \
  http://supervisor/addons/<slug>_ebusd/options
```

### 4. Install these definitions

The add-on's config folder is `/addon_configs/<slug>_ebusd/` on the host and
`/config/` inside the container. Copy the `config/` directory of this repository
there, so the files end up at:

```
/addon_configs/<slug>_ebusd/ebusd/tem/15.csv
/addon_configs/<slug>_ebusd/ebusd/tem/_templates.csv
/addon_configs/<slug>_ebusd/ebusd/tem/broadcast.csv
```

Restart the add-on. A clean start looks like this:

```
[main notice] ebusd 26.1 started on device: ..., serial speed, enhanced
[bus notice] signal acquired
[bus notice] scan 15: ;TEM;22420;0502;0102
[mqtt notice] connection established
[main notice] found messages: 63 (0 conditional on 0 conditions, 44 poll, 4 update)
```

No `error reading config files` line. Entities appear under the MQTT integration
within a few minutes.

---

## Verifying and debugging

Everything below runs from the **Advanced SSH & Web Terminal** add-on.

**Is a message being read at all?**

```bash
curl -s "http://<slug>-ebusd:8889/data/ochsner" | python3 -c "
import json,sys
for n,m in sorted(json.load(sys.stdin)['ochsner']['messages'].items()):
    print(('SILENT ' if m.get('lastup',0)==0 else 'ok     ')+n)"
```

`lastup: 0` means never received. Give it a full poll cycle before drawing
conclusions.

**Force a single read:**

```bash
curl -s 'http://<slug>-ebusd:8889/data/ochsner/Warmwassertemp.SOLL.05.051?required&verbose'
```

**Which writable messages are registered:**

```bash
docker exec $(docker ps --filter name=ebusd --format '{{.ID}}') ebusctl find -w
```

Empty output means the write rows are malformed — see problem 4 above.

**Write a value:**

```bash
docker exec $(docker ps --filter name=ebusd --format '{{.ID}}') \
  ebusctl write -c ochsner Warmwassertemp.SOLL.05.051 51
docker exec $(docker ps --filter name=ebusd --format '{{.ID}}') \
  ebusctl read -f -c ochsner Warmwassertemp.SOLL.05.051
```

Confirm on the OTE panel that the value actually landed in the controller and
not just in ebusd's cache.

### Decoding a wrong value

When a number disagrees with the OTE display, pull the raw frame:

```bash
curl -s 'http://<slug>-ebusd:8889/data/ochsner/<message>?required&raw' | grep -o '"slave": \[[^]]*\]'
```

The slave reply is laid out as
`[length, group, value, type, unit, max_lo, max_hi, min_lo, min_hi, val…]`.
Two things to check:

- **`group`** is the parameter number the controller actually answered with.
  Values ≥ 128 belong to category 01 (subtract 128), below that to category 00.
  This is what exposes the off-by-one described above.
- **the trailing bytes** are the value. Little-endian `UIN` for flow rates and
  energy; 4-byte BCD for counters — `b0 + b1·100 + b2·10⁴ + b3·10⁶`.

Worked example — operating hours, `[10, 81, 129, 29, 0, 2, 0, 8, 96, 48, 2]`:
group 81 → parameter 02-081, tail `8,96,48,2` → `8 + 9600 + 480000 + 2000000` =
2489608, divided by 100 = **24896.08 h**, matching the display exactly.

---

## A dashboard card

The values are plain sensors, so any card works. This markdown card folds the
three OTE display screens — heat pump, DHW circuit, heating circuit — into one
compact table, which reads better from across a room than three separate cards.

```yaml
type: markdown
grid_options:
  columns: full
content: >-
  {%- set p = 'sensor.heating_ebusd_ochsner_' -%}
  {%- macro t(n) %}{{ states(p ~ n)|float(0)|round(1) }}{% endmacro -%}
  | 🔥 **Heat pump** | **{{ states(p ~ 'status_warmwasser_02_052_status') }}** |
  |:--|--:|
  | DHW is / set | {{ t('warmwassertemp_aktuell_00_004_temperature') }} / {{ t('warmwassertemp_soll_05_051_temperature') }} °C |
  | Flow / return | {{ t('vorlauftemp_waermepumpe_twv_00_007_temperature') }} / {{ t('ruecklauftemp_waermepumpe_twr_00_008_temperature') }} °C |
  | Source in / out | {{ t('eintrittstemp_waermequelle_tqe_00_071_temperature') }} / {{ t('austrittstemp_waermequelle_tqa_00_070_temperature') }} °C |
  | Buffer top / mid | {{ t('vorlauftemp_puffersp_tpo_00_015_temperature') }} / {{ t('vorlauftemp_puffersp_tpm_00_017_temperature') }} °C |
  | Heating flow | {{ t('sollwert_heizkreis_vorlauftemp_01_002_temperature') }} °C |
  | 🏠 Room is / set | {{ t('raumtemperatur_00_001_temperature') }} / {{ t('istwert_heizkreis_vorlauftemp_00_002_temperature') }} °C |
  | 🌤️ Outside now / mean | {{ t('aktuelle_aussentemperatur_00_000_temperature') }} / {{ t('aussentemperatur_mittelwert_02_020_temperature') }} °C |
```

Note the labels: `sollwert_heizkreis_vorlauftemp_01_002` really is the heating
**flow** temperature and `istwert_heizkreis_vorlauftemp_00_002` really is the
room **setpoint** — the entity IDs keep the upstream names, which are shifted.
See problem 1.

`05-051` is the configured DHW setpoint, constant at whatever you set.
`01-004` is the momentary demand and drops to 10 °C when no charge is
requested — useful to see whether a charge cycle is active, misleading if
you expect the setting.

Entity IDs are derived from the message names, so if you rename messages in
`15.csv` the entity IDs follow. Check yours first:

```jinja
{% for s in states.sensor if 'ochsner' in s.entity_id %}{{ s.entity_id }} = {{ s.state }}
{% endfor %}
```

To make the hot-water setpoint adjustable from the dashboard, expose it as a
`number` entity through MQTT and point its command topic at ebusd's write topic;
alternatively call `ebusctl write` from a shell command. Both work — the first
is tidier, the second is quicker to get going.


---

## Known limitations

**Some entities are not created automatically.** Home Assistant discovery is
driven by the add-on's `mqtt-hassio.cfg`, which maps field name and unit to an
entity class. Temperatures, statuses and percentages come through; flow rates
(`L/min`), energy counters (`kW/h`, `MW/h`) and the BCD counters do not. Until
that mapping is extended, define them by hand — the raw JSON is published on
`ebusd/ochsner/<message>`:

```yaml
mqtt:
  sensor:
    - name: "Ochsner flow rate heating"
      state_topic: "ebusd/ochsner/Volumenstrom.Waermenutzung.21.002"
      value_template: "{{ value_json.lmin.value }}"
      unit_of_measurement: "L/min"
      state_class: measurement
      unique_id: ochsner_flow_heating
```

**Heating-curve parameters report broken ranges.** `Norm.Aussentemperatur.03.012`
and `Vorlauftemp.Norm.Aussentemp.03.013` return a minimum above their maximum.
The values themselves look plausible; the scaling is not solved. They are not
needed in summer, so this is untested rather than fixed.

**`Warmwassertemp.Spar.05.086` reads 0.** Either absent on this firmware or
subject to the same shift. Not investigated.

---

## Writable registers

`ebusctl find -w` lists 12. The two that matter for PV-surplus control:

| Message | Range | Effect |
|---|---|---|
| `Warmwassertemp.SOLL.05.051` | 10–55 °C | Raise it above the current tank temperature and the heat pump starts a DHW cycle on its own |
| `Betriebswahl.Waermepumpe.09.075` | 0 = off, 1 = automatic | Enables or blocks the heat generator |

There is **no** direct "start the compressor" command on the bus, and that is
deliberate: minimum off-time, defrost and pressure limits all live in the
controller. Raising a setpoint is a request; the controller decides when to act
on it. That is the safe way to do this, and it is enough — a surplus automation
that lifts the DHW setpoint from 50 to 55 °C turns the tank into a heat store
without ever touching compressor logic.

A word of caution on everything else: the parameters behind the OTE service
password — compressor limits, defrost logic, safety cut-outs — are writable over
eBUS because eBUS has no authentication at all. A wrong value there can damage
the machine, and you may not notice for months. Read freely; write only what you
understand.

---

## Credits

- [wiedwo](https://github.com/wiedwo/ebusd-guide-Ochsner-GMWW-with-Home-Assistant)
  — the GMWW configuration this is derived from, and the step-by-step guide
- [Lorilatschki](https://github.com/Lorilatschki/ebusd-guide) and
  [cybersmart-eu](https://github.com/cybersmart-eu/ebusd-ochsner) — earlier
  Ochsner work
- [bensch2293](https://github.com/bensch2293/Ochsner-ebus) — a second reference
  configuration for OTE plus
- [john30](https://github.com/john30/ebusd) — ebusd and the adapter hardware
- [LukasGrebe](https://github.com/LukasGrebe/ha-addons) — the Home Assistant
  add-on

Discussion for Ochsner on eBUS lives in
[john30/ebusd#922](https://github.com/john30/ebusd/discussions/922).

## Contributing

If you run a different OTE revision, the most useful thing you can contribute is
a mapping table: parameter number from the display, the message name ebusd used,
and the raw slave frame. That is what made the off-by-one visible here, and it
takes about ten minutes to collect.

## Licence

Configuration files are derived from wiedwo's repository and carry its terms.
Documentation in this repository is CC BY 4.0.

Not affiliated with or endorsed by Ochsner. Use at your own risk.
