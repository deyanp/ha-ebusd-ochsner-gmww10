# ebusd configuration files

Copy this directory to the ebusd add-on config folder so the files land at:

```
/addon_configs/<slug>_ebusd/ebusd/tem/15.csv
/addon_configs/<slug>_ebusd/ebusd/tem/_templates.csv
/addon_configs/<slug>_ebusd/ebusd/tem/broadcast.csv
```

then start ebusd with `--configpath=/config/ebusd`.

The `tem/` subdirectory name matters: ebusd picks the manufacturer folder from
the scan result (`MF=TEM`), and the `15` prefix from the slave address.
