## NEF-to-PCF smoke test

Tested on 2026-11-09 with Open Exposure revision
`ae60c8db252bd4ddc429f7ff73cc5ad2cbb1a532` and Open5GS
`v2.8.0-48-gf87da61`.

From this directory, send the sample request to NEF.

```sh
curl -i -H 'Content-Type: application/json' \
  --data-binary @qos-request.json \
  http://127.0.0.1:8089/3gpp-as-session-with-qos/v1/lab-af/subscriptions
```

With no active PDU session for the sample UE, PCF returned HTTP/2 `404`
from `/npcf-policyauthorization/v1/app-sessions` and NEF returned `500`.
For an active UE, set its address, DNN, slice and matching flow descriptions
in the request. Use a notification URL reachable from PCF.

## Amarisoft NR SA QoS signaling

The renamed `openexposure_nef` image was tested with Amarisoft gNB
`2026-06-12` and a phone with an active `internet` PDU session (SST 1).
The `qos3` request matched a single UDP flow.

- NEF returned `201` on creation and `200` on retrieval. PCF returned
  HTTP/2 `201` and sent a PCC rule and QoS decision to SMF. The UPF
  accepted the PFCP session modification that installed the PDRs and QER.
- NEF returned `204` on deletion and `404` on subsequent retrieval. PCF
  returned HTTP/2 `204` and sent the PCC rule removal to SMF. The UPF
  accepted the PFCP removal.

The Amarisoft `qos_flow_get` snapshots and NGAP/RRC traces showed the
following mapping on the same PDU session.

| Stage | Default flow | Dedicated flow |
| --- | --- | --- |
| Before create | QFI 1, 5QI 9, DRB 1 | Absent |
| Subscription active | QFI 1, 5QI 9, DRB 1 | QFI 4, 5QI 1, DRB 2 |
| After delete | QFI 1, 5QI 9, DRB 1 | Released |

The phone completed RRC reconfiguration for both the addition and release
of DRB 2. While the flow was active, the gNB reported 4 Mbps GBR and MBR
in both directions. Packet-level QFI marking and throughput enforcement
remain unverified.

UERANSIM can be used for the connectivity check. The radio test requires
PDU session modification, which [UERANSIM's feature set](https://github.com/aligungr/UERANSIM/wiki/Feature-Set)
does not include.
