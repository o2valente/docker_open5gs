## NEF-to-PCF test

Tested with Open Exposure revision `ae60c8db252bd4ddc429f7ff73cc5ad2cbb1a532`
and Open5GS `v2.8.0-48-gf87da61` on 2026-11-08.

From this directory, send the sample request to NEF:

```sh
curl -i -H 'Content-Type: application/json' \
  --data-binary @qos-request.json \
  http://127.0.0.1:8089/3gpp-as-session-with-qos/v1/lab-af/subscriptions
```

NEF sent the policy-authorization request to PCF over HTTP/2. The sample UE
had no active session, so PCF returned 404 and NEF returned 500:

```text
NEF: Full HTTP response: ... 404 Not Found ... HTTP/2.0 ...
PCF: ERROR: Not found [/npcf-policyauthorization/v1/app-sessions]
```

This confirms the NEF-to-PCF connection. For a QoS session, update the sample
with the UE address, matching flow descriptions, DNN, slice, QoS reference,
and a callback URL reachable from PCF.

## QoS testing

End-to-end QoS changes do not work with UERANSIM: it does not implement PDU
session modification, which is needed to apply changes to an established
session. It also omits the radio layers below RRC, so it cannot demonstrate
radio scheduling or bandwidth enforcement. NEF-to-PCF connectivity can still
be checked without those procedures.
[UERANSIM feature set](https://github.com/aligungr/UERANSIM/wiki/Feature-Set).

Use Amarisoft gNB and UE software for the end-to-end QoS test. Amarisoft
supports NR 5QI handling, traffic flow templates and dedicated bearers.
[Amarisoft features](https://www2.amarisoft.com/technology).

The Amarisoft integration test is pending. Add the NEF, PCF, SMF and Amarisoft
logs after the test to show session creation, QoS changes and deletion.
The bandwidth values in `asSessionWithQos.yaml` describe requested QoS;
enforcement must be checked on the RAN and UE.
