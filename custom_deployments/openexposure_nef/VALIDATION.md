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
- The Amarisoft trace showed NGAP adding QFI 4 with 5QI 1 and RRC mapping
  it to DRB 2. The phone completed RRC reconfiguration. The gNB reported
  4 Mbps GBR and MBR in both directions.
- NEF returned `204` on deletion and `404` on subsequent retrieval. PCF
  returned HTTP/2 `204` and sent the PCC rule removal to SMF. The UPF
  accepted the PFCP removal, and Amarisoft released QFI 4 and DRB 2.
  The gNB then reported only the default QFI 1.

The test verified signaling and bearer state. Throughput enforcement remains
unverified.

UERANSIM can be used for the connectivity check. The radio test requires
PDU session modification, which [UERANSIM's feature set](https://github.com/aligungr/UERANSIM/wiki/Feature-Set)
does not include.
