# NEF integration validation

Validated on 2026-11-08 on an Ubuntu 20.04 x86_64 VM with Docker Compose
v2.21.0. The NEF connectivity checks required no UE or RAN. The separate
VM core migration and its existing UE check are described below.

## Build and configuration

- `docker build -t docker_nef:review ./nef` succeeded using the committed
  Dockerfile, Go 1.23.10 and upstream revision
  `ae60c8db252bd4ddc429f7ff73cc5ad2cbb1a532`.
- Compose configuration validation passed locally and on the VM.
- Resolved core service definitions match the standard SA deployment, apart
  from the additional NEF environment variables inherited from `.custom_env`.
- Static checks passed for shell syntax, unique addresses within the subnet,
  build contexts, bind-mount sources and example request JSON.
- Missing `NEF_IP` or `PCF_IP` causes immediate exit with status 2 and a
  variable-specific message.
- An unreachable PCF causes exit status 1 after 30 attempts with the message
  `NEF: PCF at 127.0.0.2:7777 did not become reachable after 30 attempts`.

## PCF exchange

The new image was tested using the committed custom Compose deployment with
Open5GS `v2.8.0-48-gf87da61`, matching the repository's source pin. The official
image was pinned to
`ghcr.io/herlesupreeth/docker_open5gs@sha256:1a69372dbf1478485f546974f6d5df417fdfda0a61a609c5ce692d67577cc155`.

The test used the documented `up -d --no-build nef` dependency subset:
MongoDB, NRF, SCP, PCF and NEF. All five containers stayed running and NEF
became healthy. Test-only changes selected image tag `docker_nef:review`,
host port 18089 and separate network/volume names to avoid disturbing the
existing lab. No entrypoint or core-configuration overrides were needed.

The supplied `qos-request.json` was posted to:

```text
http://127.0.0.1:18089/3gpp-as-session-with-qos/v1/lab-af/subscriptions
```

Observed evidence:

```text
NEF: Full HTTP response: ... 404 Not Found ... HTTP/2.0 ...
     Server:[Open5GS v2.8.0-48-gf87da61]
PCF: ERROR: Not found [/npcf-policyauthorization/v1/app-sessions]
     (../src/pcf/pcf-sm.c:325)
```

Inspection of the running PCF's source confirmed this branch responds when
no matching UE session is found. NEF returned HTTP 500 with
`Error: could not create PCF Policy Authorization context`, reflecting its
upstream PCF-error handling. NEF remained running and its listener check passed.

Additional northbound checks:

| Request | Observed response |
| --- | --- |
| Unknown `qosReference` | HTTP 400, `Error: invalid QoS reference` |
| Missing UE IPv4 address, without another UE identifier | HTTP 501, `Error: not in the single or multiple UE cases` |
| GET unknown subscription ID | HTTP 404, `Error: could not find af/subId` |

## Existing VM core migration

The VM's native Open5GS core was separately migrated to Docker using the same
pinned image. Host networking preserves its SBI addresses, external N2/N3
endpoints, PLMN, slice, UE subnet, DNS and MTU. All 43 subscriber documents
were restored to Docker MongoDB and compared across every field before
restarting the existing test UE. The full set of 11 core network functions
and MongoDB remained running. The existing UE regained an active PDU session
at its original address, and gateway and Internet pings passed.

VM-specific configuration and subscriber credentials remain on the VM under
`~/open5gs-docker-migration`; they are not part of this contribution. Native
configuration, controllers and database snapshots are retained there for
rollback. The existing lab's core controller now delegates to Docker.

Successful QoS authorization and enforcement remain outside this connectivity
check. The standard custom deployment's full RAN-facing topology was not
tested; the documented NEF/PCF dependency subset was tested directly.
