# Docker Image Analysis

## Objective

The original image is approximately **1.2 GB**, runs as `root`, and contains **47 known CVEs**. The replacement uses a pinned Alpine Node.js base, a multi-stage build, production-only dependencies in the runtime stage, UID/GID 1001, a restricted build context, and a container health check. Vulnerability counts must be confirmed by scanning the built image; no Dockerfile can guarantee zero CVEs indefinitely.

## Estimated Image Size Reduction

A naive `FROM node:latest` image can contain a full OS distribution, build tooling, development dependencies, tests, caches, and source-control files.

A realistic target for this Alpine multi-stage image is approximately **150–300 MB**, depending on production dependencies. Compared with the original 1.2 GB image, this is roughly a **75–88% reduction**. The exact result must be measured after building.

```bash
docker build -t secure-node-app:1.0.0 .
docker images secure-node-app:1.0.0
```

## Multi-Stage Build and Attack Surface

The build stage may contain compilers, headers, development dependencies and source code. The runtime stage receives only:

- `package.json`
- production `node_modules`
- `dist/`
- the Node.js runtime

Build tooling and development dependencies therefore do not become part of the production image. Fewer packages and executables mean a smaller attack surface and fewer components that can contain vulnerabilities.

## Non-Root Execution

The runtime image creates:

```text
appuser
UID/GID: 1001
```

and uses:

```dockerfile
USER 1001:1001
```

A compromised process therefore does not start with root privileges. This limits the privileges available to an attacker and reduces the impact of application-level compromise. It is defence in depth, not a replacement for other controls.

## `.dockerignore`

The build context excludes `node_modules`, `.git`, tests, coverage, local environment files, IDE metadata, caches and CI metadata. This reduces build-context size and prevents unnecessary or sensitive local files from entering the build context.

## CI Image Scanning

I would use **Trivy** in CI:

```bash
trivy image --severity HIGH,CRITICAL --exit-code 1 secure-node-app:1.0.0
```

The scan should cover both OS packages and application dependencies. The CI policy should define which findings block release.

## CVE With No Available Fix

I would first determine whether the affected package is present in the final runtime image and whether the vulnerable functionality is reachable or used.

Then I would:

1. Check for a newer supported base image or alternative minimal base.
2. Remove the affected component if it is not required.
3. Apply compensating controls such as non-root execution, restricted networking, dropped capabilities and a read-only filesystem where compatible.
4. Document the CVE, exploitability assessment, compensating controls and accepted risk.
5. Monitor the upstream project and rebuild when a fix becomes available.

The finding should not simply be suppressed without an exploitability and risk assessment.

## Verification

Build:

```bash
docker build -t secure-node-app:1.0.0 .
```

Run:

```bash
docker run --rm -p 3000:3000 secure-node-app:1.0.0
```

Verify the user:

```bash
docker run --rm secure-node-app:1.0.0 id
```

Expected UID/GID: `1001`.

Verify health:

```bash
curl http://localhost:3000/health
```

Scan:

```bash
trivy image --severity HIGH,CRITICAL secure-node-app:1.0.0
```

## Additional Production Hardening

Where supported by the orchestrator, I would also use:

- Read-only root filesystem
- Dropped Linux capabilities
- No privileged mode
- CPU and memory limits
- Restricted network access
- Image signing/verification
- Continuous vulnerability scanning

## Key Decisions

| Decision | Benefit |
|---|---|
| Alpine base | Smaller runtime footprint |
| Pinned version | Reproducible builds |
| Multi-stage build | Removes build tooling |
| `npm ci` | Lock-file based dependency installation |
| `npm prune --omit=dev` | Removes development dependencies |
| UID/GID 1001 | Prevents root execution |
| `.dockerignore` | Smaller and safer build context |
| HEALTHCHECK | Detects unhealthy containers |
| Trivy | Detects known vulnerabilities |

## Conclusion

The design reduces image size and attack surface without adding unnecessary complexity. Security is layered across the image, dependency management, runtime identity, vulnerability scanning and deployment-time hardening.
