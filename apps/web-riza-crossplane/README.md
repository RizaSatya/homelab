# web-riza-crossplane

`web-riza-crossplane` is a parallel proving-ground app for the Crossplane-backed
`WebApp` golden path.

That path is the stable contract used by the tenant app `ApplicationSet`:

- `apps/web-riza-crossplane/app`

This app intentionally leaves the existing `web-riza` deployment untouched while
reusing the same application image through the platform API.
