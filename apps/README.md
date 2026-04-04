# Tenant Apps

This directory holds tenant workload entrypoints for the homelab platform.

Current contract:

- a tenant app lives at `apps/<app-name>/app/`
- the `homelab-apps` `ApplicationSet` discovers that path automatically
- the generated Argo CD `Application` name and destination namespace both match `<app-name>`

Current tenant apps:

- `web-riza`
- `garmin-scraper`

Future Backstage onboarding and platform APIs should target this contract.
