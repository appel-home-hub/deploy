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
| `TZ` | time-date | `America/Chicago` |

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

2. Push. Once the Action is green, `ghcr.io/appel-home-hub/<component>` exists.
3. Add a service to `compose.yaml` here, using `image: ghcr.io/appel-home-hub/<component>:latest`, plus any new env vars in `.env.example`, the table above, and the Portainer stack.
4. In Portainer, choose **Pull and redeploy**.
5. Stop the old container in the legacy `components/` project.

## Updating

In Portainer, open the `home-hub` stack and choose **Pull and redeploy**, with "Re-pull image" turned on. Only containers whose image changed are recreated.

To make updates automatic later, add a Portainer stack webhook and call it from the last step of `build-image.yml`, with the URL stored as an org secret. Nothing else needs to change.

## One-time setup

1. **Make `appel-home-hub/mqtt-client` public**, so Actions can check out the submodule without a token.
2. **Share the workflow:** in this repo, go to Settings → Actions → General → Access and choose *Accessible from repositories in the 'appel-home-hub' organization*.
3. **Portainer registry:** go to Registries → Add registry → GitHub Container Registry. Use your GitHub username and a **classic** PAT with only the `read:packages` scope; GHCR doesn't accept fine-grained tokens for pulling.
4. **Portainer stack:** go to Stacks → Add stack → Repository.
    - Repository URL: `https://github.com/appel-home-hub/deploy`
    - Compose path: `compose.yaml`
    - Authentication: a fine-grained PAT with Contents: read on this repo
    - Environment variables: from the table above

## Run the stack locally

```bash
cp .env.example .env
docker login ghcr.io          # username + classic PAT with read:packages
docker compose up -d
```
