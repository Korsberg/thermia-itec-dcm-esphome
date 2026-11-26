# Home Assistant views

Two ready-made views for the sensors this firmware exposes. Menu and label texts are in Swedish, like the pump's own menus.

| File | What it shows |
|---|---|
| [graphs-view.yaml](graphs-view.yaml) | Degree-minutes with the pump's own START, STOP and additional heat limits, compressor frequency with defrost, defrost counters |
| [esp32-status-view.yaml](esp32-status-view.yaml) | ESP32 health: memory, chip, network, bus and gateway status |

## Requirements

- Home Assistant dashboard in the newer "sections" layout.
- Custom cards from HACS: [apexcharts-card](https://github.com/RomRider/apexcharts-card) (graphs) and [mushroom](https://github.com/piitaya/lovelace-mushroom) (status view).

## Add a view

1. Open your dashboard, click the pencil (edit).
2. Add a view (the **+** next to the tabs) and choose to edit it in YAML, or open the raw configuration editor from the three-dot menu.
3. Paste the file's content and save.
4. If a chart shows `N/A` or `Loading...`, do a hard reload of the browser page.

## Change the entity names to yours

The entity ids in these files start with `pannrummet_thermia_itec_xt10_s3_thermia_`. `pannrummet` is the name of the area in the author's installation and `thermia_itec_xt10_s3` comes from the device name. Home Assistant builds the id from the area, the device name and the sensor name, so yours will differ.

To find yours, paste this in Developer tools, Template:

```
{% for e in states | map(attribute='entity_id') | select('search','gradminuter|meny_varme_start|avfrostning|compressor_frequency') %}{{ e }}
{% endfor %}
```

Then replace the common prefix (`pannrummet_thermia_itec_xt10_s3_`) in the file with yours.

## Notes

- The START, STOP and additional heat limit lines are the pump's own settings (`0431`, `0432`, `0457`). They show a flat line from the moment the ESP has read them after a restart.
- Several sensors are marked "kandidat" in their names. The defrost counter and the two defrost flags have been verified, see [docs/defrost.md](../docs/defrost.md) and [docs/sensors.md](../docs/sensors.md).
- The Enhetsinfo tile is left out of the status view because that text sensor is too long for Home Assistant to keep as a state.
