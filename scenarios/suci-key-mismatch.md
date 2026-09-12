# Scenario: SUCI Key Mismatch (Authentication MAC Failure)

## Summary

A UE with a valid, provisioned SUCI/SUPI attempts registration using a corrupted permanent key (K). The AMF and AUSF correctly identify the subscriber by SUCI but reject the authentication when the UE's computed MAC does not match the expected value. This scenario documents the forensic signature of a key mismatch, distinguishing it from an unprovisioned subscriber failure (see `suci-deconcealment.md` for that comparison).

## Why this matters

A K mismatch in a live network could indicate several things worth distinguishing forensically: a cloned or corrupted SIM/USIM, a subscriber database sync issue between core network elements, or an active attack attempting credential stuffing against a known, valid identity. The specific log signature this scenario produces (`Authentication failure(MAC failure)`) is diagnostically different from an unknown or unprovisioned subscriber, and a SOC analyst should be able to tell the two apart at a glance in production logs.

## Environment

- Open5GS core (AMF, AUSF, UDM, and supporting NFs) running on Ubuntu 26.04 LTS
- UERANSIM v3.3.0 simulating gNB and UE
- Test subscriber: IMSI `999700000000001`, provisioned K `465B5CE8B199B49FAA5F0A2EE238A6BC`, OPc `E8ED289DEBA952E4283B54E88E6183CA`, default slice SST 1, DNN `internet`

## Reproduction steps

1. Confirm baseline registration succeeds using the unmodified subscriber config (`config/open5gs-ue.yaml`) against the running core. This establishes the clean reference state.
2. Copy the working UE config to preserve the original:
   ```
   cp config/open5gs-ue.yaml config/open5gs-ue-suci-test.yaml
   ```
3. Edit the copy and change a single character in the `key` field, leaving every other field (including SUPI/IMSI and OPc) untouched:
   ```
   key: '465B5CE8B199B49FAA5F0A2EE238A6BD'
   ```
4. With the gNB already running and connected to the AMF (`NG Setup procedure is successful`), start the UE against the corrupted config:
   ```
   sudo build/nr-ue -c config/open5gs-ue-suci-test.yaml
   ```
5. Observe the UE-side output and the AMF logs (`sudo journalctl -u open5gs-amfd -f`) in parallel.

## Evidence: AMF log output

```
[amf] INFO: [suci-0-999-70-0000-0-0-0000000001] known UE by SUCI
[gmm] INFO: Registration request
[gmm] INFO: [suci-0-999-70-0000-0-0-0000000001]    SUCI
[amf] WARNING: GUTI has already been allocated
[gmm] WARNING: [suci-0-999-70-0000-0-0-0000000001] Authentication failure [20]
[gmm] WARNING: Authentication failure(MAC failure)
[amf] WARNING: [suci-0-999-70-0000-0-0-0000000001] Authentication reject
[gmm] WARNING: [imsi-999700000000001] Failure in transaction; restoring context and transitioning to REGISTERED.
```

## Analysis

The AMF recognizes the subscriber (`known UE by SUCI`), meaning the SUCI itself decrypts correctly and maps to a real, provisioned identity in the UDM. This is the critical distinguishing point: the failure occurs one step later, during the 5G-AKA authentication exchange, when the AUSF/UDM validate the MAC computed from the UE's key against the expected value derived from the provisioned K. A mismatched K produces a MAC that does not verify, which the network reports plainly as `Authentication failure(MAC failure)`, followed by an explicit `Authentication reject`.

This is forensically distinct from a `Cannot find SUCI [404]` error, which indicates the network has no record of the subscriber at all (unprovisioned, deleted, or possibly a spoofed/malformed identity). A MAC failure instead points to a real, known subscriber whose credential material does not match what the network expects, consistent with a cloned or corrupted SIM/USIM, a provisioning error between subscriber databases, or a deliberate attempt to authenticate as a known identity without the correct key.

The `GUTI has already been allocated` warning is a secondary artifact worth noting: it appears because the AMF retained context from an earlier legitimate registration of this same subscriber. In a production environment, this same pattern (a known GUTI context colliding with a failed authentication attempt) could be a useful correlation point when investigating repeated authentication failures against a single identity, since it shows the network already trusted this subscriber recently.

## What a SOC analyst should do

On seeing `Authentication failure(MAC failure)` tied to a specific SUCI or IMSI in AMF logs:

- Correlate against how many failed attempts have occurred for that identity in a given window; a single failure may be a legitimate SIM fault, repeated failures against the same identity suggest an active attempt to authenticate without valid credentials.
- Cross-reference with the UDM/subscriber database to confirm the provisioned K has not been altered or desynchronized between core network elements.
- If the subscriber reports their device is behaving normally and this is unexpected, this pattern combined with unexplained failures may warrant investigating for SIM cloning.

## Open questions for further testing

- Does corrupting the OPc instead of K produce an identical `MAC failure` signature, or a distinguishable one?
- Does repeated rapid failure against the same SUCI trigger any rate limiting or additional logging at the AMF or AUSF that single-attempt testing does not surface?
