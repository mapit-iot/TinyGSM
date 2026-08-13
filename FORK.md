# Mapzon fork of TinyGSM

**Upstream**: [vshymanskyy/TinyGSM](https://github.com/vshymanskyy/TinyGSM)
**Base**: tag `v0.12.0` (`675b4dd2bf6b02c64648dbc19bb0e417d882760d`)
**Branch**: `mapzon-0.12.0`
**Consumer**: [mapit-iot/mapit-m510-fw](https://github.com/mapit-iot/mapit-m510-fw) — `lib/modem_tinygsm`, SIM7080G LTE-M modem driver for the MapIT M510 tracker.

This fork exists to carry a small number of deviations that cannot be expressed
from outside the library. TinyGSM dispatches URC handling through CRTP
(`thisModem().handleURCs()`, `TinyGsmModem.tpp`), so no subclass, macro, or glue
layer can intercept it — a source change is the only mechanism available.

Everything not listed below is upstream `v0.12.0`, unmodified. Verify with:

```sh
git diff v0.12.0..mapzon-0.12.0
```

## Deviations

### 1. `SMS Ready` URC no longer triggers a hidden re-initialization (2026-08-13)

**File**: `src/TinyGsmClientSIM7080.h` (`handleURCs`, constructor)

Upstream reacts to the module's `SMS Ready` reset announcement by re-entrantly
calling `init()`:

```cpp
} else if (data.endsWith(GF(AT_NL "SMS Ready" AT_NL))) {
  data = "";
  DBG("### Unexpected module reset!");
  init();
  data = "";
  return true;
}
```

`handleURCs` runs from inside `waitResponse()`, i.e. while a host command is in
flight. The nested `init()` then:

- runs its own `AT` ping loop and consumes the reply the outer caller was
  waiting for — the caller times out with an empty response;
- issues `AT+CMEE=0`, silently reverting a host-configured `AT+CMEE=2`, so all
  subsequent error reporting loses its detail;
- issues `ATE0`, `AT+CLTS=1`, `AT+CBATCHK=1`, `AT+CPIN?` outside the host's
  provisioning sequence.

On the M510 this reproduced as a first-`init()` failure on every cold boot (the
`SMS Ready` URC lands ~3.5 s after PWRKEY, inside the provisioning window) and,
more dangerously, as an invisible mid-session re-provision whenever the module
browned out or reset.

The fork records the event instead:

- `bool unexpected_reset_` / `uint32_t unexpected_reset_at_` members, cleared in
  the constructor;
- `handleURCs` sets them (and keeps the existing `DBG` line) rather than calling
  `init()`;
- `bool consumeUnexpectedReset()` — consume-on-read — and
  `uint32_t lastUnexpectedResetMillis()` expose it to the host driver, which
  re-provisions deliberately, at a point where it owns the modem, and re-asserts
  its own `CMEE` setting.

The members carry no internal synchronization: they are written from whichever
task is driving the modem and are meant to be read under the same serialization
the host already applies to modem access (on the M510, a recursive modem mutex).

**Behavioral note for other users of this fork**: an application that relied on
the automatic recovery must now call `consumeUnexpectedReset()` and run its own
re-initialization. Ignoring the flag leaves the module un-reinitialized after a
reset.

## Rules for future deviations

1. **No `handleURCs` pattern may be a prefix of any `r1`..`r7` string used
   anywhere.** `handleURCs` matches per character with `String::endsWith`, so a
   short pattern fires before a longer response string can complete. Adding
   `+APP PDP:`, for example, would shadow `+APP PDP: 0,ACTIVE` — the response
   both `gprsConnect()` and the M510 PDP activation path wait for — and break
   PDP activation outright. Same trap class: `+CPIN:`.
2. **No changes to TLS, certificate handling, cipher configuration, or
   `CSSLCFG`/`CASSLCFG` semantics.** TLS policy lives on the application side.
3. **All received bytes are untrusted.** Any parsing added here does bounded
   pattern matching and bounded integer extraction only.
4. Every deviation is marked in-source with a `MAPZON FORK` comment and a date,
   and listed in this file.

## Rebasing onto a newer upstream

1. `git fetch upstream --tags`
2. Branch `mapzon-<new-version>` from the new tag, re-apply the deviations above.
3. Re-run the diff review (`git diff <new-tag>..mapzon-<new-version>`) and
   confirm it contains nothing but the listed deviations.
4. Re-pin the consumer by full commit SHA (never by branch or tag — both can be
   re-pointed).

## License

TinyGSM is licensed **LGPL-3.0**; this fork keeps that license unchanged (see
`LICENSE`). Modified files carry a change note in-source as required, and this
repository is public so that the modified library source is available to
recipients of any binary that links it.
