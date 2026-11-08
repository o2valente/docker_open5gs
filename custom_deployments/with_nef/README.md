# Open Exposure NEF with Open5GS

This optional 5G SA deployment adds the Open Exposure **AS Session with QoS**
service to the existing Open5GS core. NEF connects directly to the PCF over
HTTP/2 at `http://${PCF_IP}:7777`. The northbound API uses HTTP/1.1.

The integration covers NEF-to-PCF policy-authorization connectivity. A PCF
rejection for a missing UE session is sufficient to demonstrate the exchange;
successful QoS authorization requires a matching active session. No RAN or UE
deployment is required for this connectivity check.

Upstream references:

- [AS Session with QoS service](https://gitlab.eurecom.fr/open-exposure/nef/as-session-with-qos)
- [Core compatibility and integration guide](https://gitlab.eurecom.fr/open-exposure/nef/deployment/-/blob/develop/INTEGRATING.md)

See [validation results](VALIDATION.md) for the tested versions, observed
responses and limits of the deployment check.

## Build and configure

Follow the root [README](../../README.md) for Docker prerequisites and building
or pulling the `docker_open5gs` image. Build NEF from the repository root:

```sh
docker build -t docker_nef ./nef
cd custom_deployments/with_nef
```

The Dockerfile pins Open Exposure to
`ae60c8db252bd4ddc429f7ff73cc5ad2cbb1a532` and uses Go 1.23.10. It builds
independently of Open5GS. The existing image workflow also builds `docker_nef`;
registry pulls will be available after the repository maintainer publishes it.

Use **this directory's `.custom_env`** for this deployment. Adjust
`DOCKER_HOST_IP` and the core network settings for your host as described in
the root README. The NEF-specific defaults are:

| Variable | Default | Purpose |
| --- | --- | --- |
| `NEF_IP` | `172.22.0.43` | Container address on `TEST_NETWORK` |
| `NEF_BIND_IP` | `127.0.0.1` | Host interface for API access |
| `NEF_BIND_PORT` | `8089` | Published API port |
| `PCF_IP` | `172.22.0.27` | Open5GS PCF address; SBI port is 7777 |

For remote lab access, set `NEF_BIND_IP` to the host's reachable interface IP
and restrict access to trusted clients. CAPIF registration and authentication
are disabled. This is a plain-HTTP lab configuration.

The mounted [NEF template](../../nef/asSessionWithQos.yaml) defines `qos1` through
`qos4`, with example bandwidths of 1, 2, 4 and 8 Mbps. Change these mappings to
match your policies. They do not guarantee data-plane QoS enforcement.
Initialization renders a separate runtime configuration without modifying the
mounted template. It waits up to 30 connection attempts for the PCF listener.

## Start

From this directory, validate the manifest and start only NEF and its PCF
dependencies for a connectivity check:

```sh
docker compose --env-file .custom_env -f sa-deploy.yaml config --quiet
docker compose --env-file .custom_env -f sa-deploy.yaml up -d --build nef
docker compose --env-file .custom_env -f sa-deploy.yaml ps
docker compose --env-file .custom_env -f sa-deploy.yaml logs --tail 100 nef pcf
```

To start the full SA core instead, omit `nef` from the `up` command. This
deployment reuses the repository's container, network and volume names. Stop
any other deployment using those names before starting this one.

NEF health reports whether its API listener is accepting connections. It does
not prove PCF policy authorization works.

## Check NEF-to-PCF communication

The supplied `qos-request.json` uses a test UE address and an example callback
at the default Docker bridge gateway. For a connectivity-only check, choose a
UE address that has no active session. For actual authorization, replace it
and the flow destination with the real UE address, match DNN and S-NSSAI to
the session, and provide a working callback endpoint reachable **from PCF**.
`localhost` in a callback URL means the PCF container itself.

Run on the Docker host (adjust the URL if you changed the published port):

```sh
curl --max-time 15 -i \
  -H 'Content-Type: application/json' \
  --data-binary @qos-request.json \
  http://127.0.0.1:8089/3gpp-as-session-with-qos/v1/lab-af/subscriptions

docker compose --env-file .custom_env -f sa-deploy.yaml logs --since 2m nef pcf
```

Verify PCF received a `POST` for `/npcf-policyauthorization/v1/app-sessions`.
Without a matching session, PCF should reject the authorization; the pinned NEF
maps PCF failures to HTTP 500 and logs the PCF HTTP response. A response alone
is insufficient evidence: correlate it with PCF logs. Transport errors,
malformed requests and NEF crashes are failures, not successful connectivity.

If packet-level confirmation is needed, capture the cleartext SBI exchange on
the Docker host while sending the request (requires host capture permissions):

```sh
sudo tcpdump -i any -s 0 -w /tmp/nef-pcf.pcap 'host 172.22.0.27 and tcp port 7777'
```

Stop the capture with Ctrl-C and inspect the HTTP/2 stream in Wireshark.
Use the configured `PCF_IP` if it differs from the default.

With a matching active session, a successful create returns HTTP 201 and a
`Location` header. Use its subscription ID for retrieve and delete:

```sh
SUBSCRIPTION_ID='<ID returned by the create request>'
curl --max-time 15 -i "http://127.0.0.1:8089/3gpp-as-session-with-qos/v1/lab-af/subscriptions/$SUBSCRIPTION_ID"
curl --max-time 15 -i -X DELETE "http://127.0.0.1:8089/3gpp-as-session-with-qos/v1/lab-af/subscriptions/$SUBSCRIPTION_ID"
```

Retrieve returns HTTP 200; delete returns HTTP 204 on success. Collection
listing and PUT/PATCH updates are not implemented by this pinned upstream
service. Subscriptions are stored in NEF memory; restarting NEF loses its
local subscription state and may leave PCF application sessions needing
cleanup. Delete successful subscriptions before stopping NEF.

## Troubleshooting and shutdown

- Startup exits with a missing variable: check `.custom_env` and the command's
  `--env-file` argument.
- PCF wait times out: check the PCF logs, `PCF_IP`, network subnet and SBI port.
- HTTP 400 for an unknown QoS reference: use a key from `qosConfig`.
- PCF responds but rejects authorization: inspect its error and confirm the UE
  session, DNN, slice and configured policies if successful authorization is needed.
- API healthy but no PCF request: check the request body and NEF logs. A local
  validation error does not test southbound connectivity.

```sh
docker compose --env-file .custom_env -f sa-deploy.yaml down
```

This preserves the existing named database volumes. Other NEF APIs, NRF
discovery, CAPIF, OpenOP/CAMARA and QoS enforcement testing are outside this
deployment's scope.
