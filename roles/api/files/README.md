# roles/api/files/

Drop your house photo here as `house.jpg` (a Google Maps screenshot
works well -- north-up, with the house roughly centred) before running
`ansible-playbook site.yml --limit api`. `roles/api/tasks/app.yml`'s
"Deploy the house photo for the almanac compass panel" task copies it to
`api_house_image_remote_path` (`/etc/sensors-api/house.jpg` by default)
and fails the run with a clear error until this file exists.

If your image isn't north-up or the house isn't centred, tune
`api_compass_north_offset_deg` / `api_compass_center_x_pct` /
`api_compass_center_y_pct` / `api_compass_radius_pct` in
`inventory/group_vars/api/vars.yml` -- render `.../compass.svg` directly
(no Grafana needed) to iterate quickly; see
`sensors-apps/astronomy/README.md`.
