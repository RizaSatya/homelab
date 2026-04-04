# garmin-scraper

Live GitOps manifests for this tenant app now live under `app/`.

That path is the stable contract used by the tenant app `ApplicationSet`:

- `apps/garmin-scraper/app`

This app directory can later grow additional non-runtime metadata such as Backstage catalog files or app-specific docs without changing the runtime entrypoint.
