# Secure Node.js Docker Image

This repository contains the Docker hardening deliverables for a Node.js web application.

## Deliverables

- `Dockerfile` - production-grade, pinned Alpine Node.js image using a multi-stage build.
- `.dockerignore` - excludes dependencies, source-control metadata, tests, local configuration, and development artifacts from the build context.
- `IMAGE-ANALYSIS.md` - documents image-size reduction, attack-surface reduction, non-root execution, CI scanning, and CVE handling.

## Security controls

- Pinned Node.js and Alpine base-image versions.
- Separate build and runtime stages.
- Production dependencies only in the runtime image.
- Non-root runtime user `appuser` with UID/GID `1001`.
- Resource-friendly production environment variables.
- Container `HEALTHCHECK` for `GET /health`.

## Application assumption

The Dockerfile is written for an existing Node.js application that provides:

- `package.json` and `package-lock.json`.
- An `npm run build` script that creates `dist/`.
- A production entry point at `dist/server.js`.
- An HTTP health endpoint at `/health` on port `3000`.

Those application files are not part of this assignment repository. The required submission is limited to the Dockerfile, `.dockerignore`, and `IMAGE-ANALYSIS.md` listed above.
