# roles/api/files/

The house photo for the almanac compass panel. `MiCasa.jpg` (north-up,
house roughly centred, resized/re-encoded from the original Google Maps
screenshot -- `MiCasa-original.png` is that untouched original, kept for
reference but not deployed) is what `api_house_image_local_path` in
`inventory/group_vars/api/vars.yml` points at; `roles/api/tasks/app.yml`'s
"Deploy the house photo for the almanac compass panel" task copies it to
`api_house_image_remote_path` (`/etc/sensors-api/house.jpg` by default).

Compass geometry (`api_compass_north_offset_deg` /
`api_compass_center_x_pct` / `api_compass_center_y_pct` /
`api_compass_radius_pct`, same vars file) is already tuned against this
exact photo -- rendered and eyeballed in a browser, not guessed. If you
ever replace the photo, re-tune the same way: render `.../compass.svg`
directly (no Grafana needed) and iterate; see
`sensors-apps/astronomy/README.md`.

If you swap in a different photo, update `api_house_image_local_path`
(and `_remote_path`'s extension if the format changes) to match its
filename -- nothing requires the name `MiCasa.jpg` specifically.
