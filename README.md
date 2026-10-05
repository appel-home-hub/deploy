# deploy

The server side of Home Hub. `compose.yaml` lists every component as a pre-built image from GHCR and runs as one Portainer stack (`home-hub`).

This repo also hosts the reusable GitHub Actions workflow that every component uses to build and publish its image.

## How it fits together

```
component repo (e.g. appel-home-hub/time-date)
  push to master ──▶ .github/workflows/build.yml
                       └─ calls deploy/.github/workflows/build-image.yml
                            └─ builds the Dockerfile (with submodules)
                               └─ pushes ghcr.io/appel-home-hub/<component>:latest and :sha-<commit>

Portainer stack "home-hub" (this repo's compose.yaml)
  Pull and redeploy ──▶ pulls the :latest images ──▶ restarts changed containers
```

Image tags:

| Tag | When |
| --- | --- |
| `latest` | every push to `master` |
| `sha-<short>` | every push, so you can pin or roll back to an exact commit |
| `X.Y.Z` | pushing a `vX.Y.Z` git tag |

Pull requests build the image but don't push it.

## Environment variables

Set these on the Portainer stack. Nothing secret is committed here. See `.env.example`.

| Variable | Used by | Example |
| --- | --- | --- |
| `MQTT_HOST` | all components | `mqtt://192.168.1.9:1883` |
| `TZ` | time-date, weather | `America/Chicago` |
| `OPENWEATHER_API_KEY` | weather | your One Call 3.0 key (secret) |
| `LATITUDE`, `LONGITUDE` | weather | home coordinates (kept out of this public repo) |
| `FIREBASE_CONFIG_URL` | firebase | raw URL of the firebase config gist |
| `FIREBASE_SERVICE_ACCOUNT` | firebase | service account key JSON on one line (secret) |
| `FIREBASE_DATABASE_URL` | firebase | Realtime Database URL |
| `ECOBEE_API_KEY` | ecobee | Ecobee developer API key (secret) |
| `ECOBEE_REFRESH_TOKEN` | ecobee | initial refresh token; after the first start the container uses the rotated one saved in its `ecobee-data` volume (secret) |
| `RF_COMMANDS_URL` | radio-frequency | raw URL of the secret RF config gist (secret) |
| `RF_*` | radio-frequency | one RF code per `env:` reference in that gist (secret) |

## Adding a component

1. In the component repo, add `.github/workflows/build.yml`:

    ```yaml
    name: build

    on:
      push:
        branches: [master]
        tags: ['v*']
      pull_request:
      workflow_dispatch:

    jobs:
      image:
        uses: appel-home-hub/deploy/.github/workflows/build-image.yml@master
        permissions:
          contents: read
          packages: write
    ```

2. Push. Once the Action is green, `ghcr.io/appel-home-hub/<component>` exists. The first push creates the package as **private**: open it under the org's **Packages** tab → **Package settings** → **Change visibility** → **Public**.
3. Add a service to `compose.yaml` here, using `image: ghcr.io/appel-home-hub/<component>:latest`, plus any new env vars in `.env.example`, the table above, and the Portainer stack.
4. In Portainer, choose **Pull and redeploy**.
5. Stop the old container in the legacy `components/` project.

## Updating

In Portainer, open the `home-hub` stack and choose **Pull and redeploy**, with "Re-pull image" turned on. Only containers whose image changed are recreated.

To make updates automatic later, add a Portainer stack webhook and call it from the last step of `build-image.yml`, with the URL stored as an org secret. Nothing else needs to change.

## One-time setup

Everything here is public: this repo, `mqtt-client`, and the GHCR images. That means Portainer needs no tokens or registry entries. Secrets never go into images or this repo; they live in the Portainer stack's environment variables.

1. **Make `appel-home-hub/mqtt-client` public**, so Actions can check out the submodule without a token.
2. **Make this repo public**, so Portainer can read `compose.yaml` and every component can call the shared workflow.
3. **Portainer stack:** go to Stacks → Add stack → Repository.
    - Name: `home-hub`
    - Repository URL: `https://github.com/appel-home-hub/deploy`
    - Repository reference: `refs/heads/master`
    - Compose path: `compose.yaml` (Portainer defaults to `docker-compose.yml`)
    - Authentication: off
    - Environment variables: from the table above

### Troubleshooting

| Error | Cause |
| --- | --- |
| `error from registry: denied` | The package is still private, **or** Portainer has a stale `ghcr.io` registry entry. GHCR rejects bad credentials even for public images, so delete any GitHub or `ghcr.io` entries under Registries. |
| `open .../compose.yml: no such file or directory` | The compose path is wrong. It must be `compose.yaml`. |
| `Object not found inside the database (bucket=git_credentials …)` | The form references a deleted saved Git credential. Reload the page and leave Authentication off. |
| `workflow was not found` (in a component's Actions run) | This repo isn't public, or isn't shared with the org's repos. |

## Run the stack locally

Only do this while the server stack is stopped. Otherwise two copies of each component publish to the broker.

```bash
cp .env.example .env
docker compose up -d
```
