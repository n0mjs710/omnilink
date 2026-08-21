# IPSC-CAPABILITIES.md — Capability advertisement (reference)

**Not normative, and not needed for a working system.** Leave
`cap_safe_defaults = true` — the default — and none of this applies. The
safe defaults are values proven against real hardware, and no deployment
is yet known that needs anything else.

## Why it is exposed at all

IPSC is reverse-engineered. The defaults are what we have observed
working, not what a specification promises, so **we do not know what we
do not know.** A future repeater, console, or programming tool may need
an advertisement we do not currently send. When that happens the operator
should be able to change it from a config file rather than by patching
code — so the flags are exposed, documented, and defaulted off.

That is the entire justification. It is insurance, not a feature.

## Using them

1. Set `cap_safe_defaults = false`. While it is true, every `cap_*` key
   is ignored.
2. Set only what you actually need.
3. **Verify with a wire capture.** IPSC gives no error for a wrong
   advertisement. The symptom is a peer behaving oddly, not a rejection,
   so an unverified change is a change you cannot evaluate.

Everything here is **per-system**, in the `[[system]]` stanza, like every
other system key (CONFIG §2.4).

## The keys

### MODE byte

| key | default | meaning |
|---|---|---|
| `cap_peer_oper` | `true` | peer is operational |
| `cap_radio_mode` | `"DIGITAL"` | `NO_RADIO` \| `ANALOG` \| `DIGITAL` \| `MIXED` |
| `cap_ts1_linked` | `true` | IPSC slot 1 linked |
| `cap_ts2_linked` | `true` | IPSC slot 2 linked |

### Service flags byte 3

| key | bit | default | meaning |
|---|---|---|---|
| `cap_csbk_call` | `0x80` | `false` | CSBK / control signalling |
| `cap_rcm` | `0x40` | `false` | repeater call monitoring |
| `cap_con_app` | `0x20` | `false` | third-party console application |

### Service flags byte 4

| key | bit | default | meaning |
|---|---|---|---|
| `cap_xnl_call` | `0x80` | `false` | XNL/XCMP connected |
| `cap_xnl_master` | `0x40` | `false` | XNL master; slave when `cap_xnl_call` and not this |
| `cap_data_call` | `0x08` | `false` | data calls supported |
| `cap_voice_call` | `0x04` | `true` | voice calls supported |

### Service flags bytes 1–2

| key | default | meaning |
|---|---|---|
| `cap_mnis` | `false` | MNIS (Mobile Network Interface Service) capable |
| `cap_ip_site_freq` | `false` | IP Site Single Frequency |
| `cap_slot1_phone` | `false` | slot 1 phone patch |
| `cap_slot2_phone` | `false` | slot 2 phone patch |
| `cap_virtual_peer` | `false` | software peer, not a physical repeater |

## Derived, never configured

Three bits are set from other config and are **not** keys, because
accepting them would let a stanza contradict itself:

| bit | follows |
|---|---|
| `PKT_AUTH 0x10` | `auth_enabled` |
| `MSTR_PEER 0x01` | `mode` |
| `XNL_SLAVE 0x20` | `cap_xnl_call && !cap_xnl_master` |

## Reference implementations

**Read both; dmrlink3 leads.** It is Python, but the most mature and most
comprehensive IPSC implementation in the family, and it covers everything
that was understood when it was written. `const.py` carries the bit
masks; `config.py` builds the flag bytes.

**ipsc2hbpc adds later decoding on service-flag bytes 1–2**, found during
that work and never backported — which is why dmrlink3 leaves those two
bytes zero. It is authoritative for those two bytes and nothing else;
dmrlink3 is authoritative for the rest.
