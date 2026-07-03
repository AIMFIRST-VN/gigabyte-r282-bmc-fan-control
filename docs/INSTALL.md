# Installation & Usage

Install the layered fan profile on your Gigabyte **MZ92-FS0** BMC (R282-Z9x /
R182-Z9x) so a passive GPU actually gets cooled — near-silent at idle, ramping in
proportion to GPU load.

> ⚠️ **You are reprogramming the BMC's fan control.** Back up first (step 3), test
> on an **idle GPU**, and keep independent hardware backstops (a GPU clock-cap
> guard + the card's ~99 °C throttle). The snapshots in [`../results`](../results)
> let you revert at any time.

## 1. Prerequisites

- A Gigabyte **MZ92-FS0** server (R282-Z9x / R182-Z9x) with the AMI MegaRAC BMC.
- **BMC admin credentials** and network reachability to the BMC web UI.
- **Python 3.8+** with `requests` — `pip install requests`.
- The passive GPU installed (e.g. AMD Radeon Pro V620 / Instinct MI210, NVIDIA
  Tesla A100 / A40 / T4, Intel Data Center GPU Flex).

## 2. Set credentials (environment only — never commit them)

```bash
export BMC_HOST=bmc.example.lan       # your BMC hostname or IP
export BMC_USER=admin
export BMC_PASS='your-bmc-password'
```

## 3. Back up the current fan profile

```bash
python3 scripts/apply-fan-profile.py status                    # show the active profile
python3 scripts/apply-fan-profile.py backup stock-profile.json # snapshot everything
```

Keep `stock-profile.json` — `restore` puts it back exactly.

## 4. Build & apply your profile

The fan profile is one JSON document listing every profile (the factory ones **and**
yours); the BMC drives each fan at the **max** duty any policy on it demands, which
is how the layered control works. A working, layered example is in
[`../results`](../results) (`fankit-v3.profile.json`): GPU-current + CPU-temperature
+ system-power policies on the GPU-side fans, a staged second fan pair, and a 91 °C
all-fan safety.

> **Sensor and fan indices are specific to the MZ92-FS0** — see
> [`fan-profile.md`](fan-profile.md) for the sensor map (`12V_GPU0` current,
> `SYS_POWER`, CPU temps, and the FAN1–4 zone wiring). Adapt them if your board or
> GPU slot differs.

Typical flow — start from your backup, add/edit your custom profile in it, then:

```bash
python3 scripts/apply-fan-profile.py apply my-fan-profile.json   # write (verified)
python3 scripts/apply-fan-profile.py mode  fankit-v3             # activate by name
```

Both writes are **verified by reading them back**; the tool errors out rather than
leaving the profile half-applied.

## 5. Verify

- **Idle:** `status` shows your profile active and the fans should be near-silent.
- **Under load:** drive a GPU workload (see [`../scripts`](../scripts) for burn-in /
  ramp-test tooling) and watch the GPU-side fans ramp with `12V_GPU0` current while
  the far fans stay quiet until total power says a second GPU is present.

## 6. Revert

```bash
python3 scripts/apply-fan-profile.py restore stock-profile.json
```

The profile lives in the BMC's own NVRAM, so once applied it runs autonomously and
survives reboots — no host daemon required.

## How the write works (for the curious)

Standard Redfish reads the fan profile but rejects writes (`405` / `400`). The web
UI uses a proprietary `/api/` interface: `POST /api/session` (form login) returns a
CSRF token and sets a session cookie; every write carries the cookie **and** an
`X-CSRFTOKEN` header. `scripts/apply-fan-profile.py` reproduces exactly that. The
full reverse-engineering story is in [`blog.md`](blog.md).
