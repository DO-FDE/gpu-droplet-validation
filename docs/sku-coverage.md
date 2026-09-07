# SKU coverage loop

The ongoing process that keeps this suite current as new GPU node types ship,
and that keeps existing thresholds honest. Coverage here is what Auto-ahoy
can fold into Caviar provisioning; the suite itself stays a single public
source of truth (`run.sh` + ghcr images) with no internal fork.

## Why this exists

Every provisioned B300 / MI325X / MI350X / MI355X hypervisor is expected to
run this suite. New NVIDIA or AMD SKUs arrive regularly; without a deliberate
loop they sit unvalidated, and without a Foresight handoff they never appear
as a Caviar pass/fail badge. Threshold drift (driver/ROCm bumps, VF vs bare-
metal differences, cross-node spread) is the same loop — recalibrate, release,
notify.

## Roles

| Who | Owns |
| --- | --- |
| Suite maintainers (this repo) | Model arms, vendored confs, floors, docs, release tarball |
| Foresight / Auto-ahoy | Toggle the new `--gpu-model` in the provisioning workflow; Caviar badge |
| Hardware / fleet ops | Idle host access for calibration runs |

Notify Foresight in **`#foresight-fde-validation`**. The Auto-ahoy ↔ suite
contract is unchanged across SKUs: TAP v14 on stdout, exit `0`/`1`/`255`,
artifacts under the results dir. Adding a SKU never changes that contract —
only the `--gpu-model` value Auto-ahoy passes and the floors inside the
images.

## Loop overview

```
new SKU announced ──► additive suite change ──► calibrate on hardware
        ▲                                              │
        │                                              ▼
 threshold refresh ◄── fleet signal / driver bump   release tarball
                                                       │
                                                       ▼
                                      notify #foresight-fde-validation
                                                       │
                                                       ▼
                                      Auto-ahoy enables --gpu-model
```

## Adding a new GPU node type

Framework (compose, entrypoints, tap-reporter, images) does **not** change.
Follow the additive pattern already used for AMD SKUs and B300.

### 1. Identify the family

| Family | `--gpu-model` prefix | Threshold file | Extra artifact |
| --- | --- | --- | --- |
| NVIDIA | `nvidia-<sku>` | `containers/_lib/nvidia_models.sh` | DCGM plugin list / params in the same arm |
| AMD | `amd-<sku>` | `containers/_lib/amd_models.sh` | Vendored RVS conf at `containers/rvs/conf/<gpu-model>/rvs_level_4.conf` (and level 5 if upstream ships it) |

The conf directory name **is** the `--gpu-model` value. Copy AMD confs
verbatim from upstream `ROCmValidationSuite/rvs/conf/<MODEL>/levels/`.

### 2. Land the additive code change

**NVIDIA** — one `case` arm in `nvidia_models.sh`: model regex, per-GPU
memory MiB, NVLink/NVSwitch expectations, `NCCL_ALLREDUCE_FLOOR`,
`REQUIRES_NVSWITCH_FABRIC`, `DCGM_DIAG_*`.

**AMD** — one `case` arm in `amd_models.sh` plus the vendored RVS conf
directory. Set `EXPECTED_GPU_MODEL_REGEX`, `EXPECTED_VRAM_MIB`,
`RCCL_ALLREDUCE_FLOOR`, `RCCL_ALLTOALL_FLOOR`. Until floors are set, a full
`run.sh` flow fails fast at `rccl-tests-amd` (unset floor); RVS-only arms
(`EXPECTED_VRAM_MIB=0`, no floors) are for one-off diagnostics only — see
[k8s-standalone.md](k8s-standalone.md).

No compose or Dockerfile edits for a same-architecture SKU. Rebuild
out-of-band AMD base images (`scripts/build-rvs-base.sh`,
`scripts/build-rccl-tests-base.sh`) only when ROCm or a new ISA (`gfx*`)
requires it — see [development.md](development.md#out-of-band-base-images-infrequent-amd-only).

### 3. Calibrate on real hardware

Need SSH (or equivalent) to at least one **idle** host of the target SKU.
Prefer three hosts when the fleet has them (MI325X pattern); a single host
is acceptable for first ship with an explicit "single-host" note (MI350X
pattern).

On the host:

```bash
curl -fsSL \
  "https://github.com/DO-Solutions/gpu-droplet-validation/releases/latest/download/gpu-droplet-validation-latest.tgz" \
  | tar --no-same-owner -xz

# Use the new --gpu-model once the arm exists locally / in a candidate release.
sudo ./run.sh --gpu-model <nvidia-…|amd-…> --gpu-count 8 \
  --node-id <host> --region <region> --run-id calibrate-001
```

Record from a clean run:

| Signal | Where | How to set the gate |
| --- | --- | --- |
| Per-GPU memory / VRAM | prereqs / `nvidia-smi` or `amd-smi static --vram` | Exact expected MiB |
| Model name string | same | Regex for the SKU |
| Collective busbw@8GB | `results/*-allreduce_run*.log` / alltoall | Mean-of-3 in-place column; floor ≈ **5–6% below** the min best healthy run |
| NVIDIA fabric | `nvidia-smi` fabric state | `REQUIRES_NVSWITCH_FABRIC` |
| AMD RVS actions | vendored conf + `results/rvs.log` | Conf-agnostic parser; no per-action floors beyond RVS PASS/FAIL |

Do **not** invent floors from datasheet peak FLOPS. Collective bandwidth
floors track fabric (NVLink / Infinity Fabric), not compute clocks. When a
new SKU shares fabric with an already-calibrated sibling (e.g. MI355X vs
MI350X), seeding floors from the sibling is allowed as a starting point —
still confirm with mean-of-3 on a real host of the new SKU and document
the confirmation in the model-file comment.

### 4. Document

- `README.md` — supported-families table row.
- `docs/test-suite.md` — `## --gpu-model <sku>` section with every TAP
  point's threshold and what `not ok` means.
- Model-file comments — calibration date, host(s), min-best numbers,
  headroom rationale, single- vs multi-host caveat.

### 5. Release

```bash
scripts/release.sh          # or --dry-run first
```

One release publishes every image and the unified tarball. Auto-ahoy pulls
`…/releases/latest/…` (or a pinned tag); there is no separate internal
build.

### 6. Notify Foresight

Post in **`#foresight-fde-validation`** once the release is public. Include:

1. **SKU** and exact `--gpu-model` string (e.g. `amd-mi355x`).
2. **Release tag** (or confirmation that `latest` includes the arm).
3. Confirmation the **TAP / exit-code contract is unchanged**.
4. Ask to **enable the Auto-ahoy toggle** for that model on the relevant
   fleets (mirrors how MI350X was "ready to toggle anytime" after the suite
   shipped).
5. Link to the PR / `docs/test-suite.md` section for triage context.
6. Any caveats (single-host calibration, known VF IET quirks, etc.).

Foresight owns enabling the workflow; suite maintainers do not change
Auto-ahoy themselves.

## Keeping existing thresholds current

Same loop, smaller scope — no new `--gpu-model`, no Foresight toggle unless
behavior changes in a breaking way.

Triggers:

- New driver / ROCm / DCGM / RVS version on production images.
- Systematic false fails (floor too high) or silent passes (floor too low)
  in Caviar / Spaces artifacts.
- A second (or third) host of a previously single-host-calibrated SKU becomes
  available — remeasure cross-node spread and adjust floors if needed.
- Vendored RVS conf upstream drift for an existing SKU.

Steps: re-run calibration on idle hardware → update the model arm comment +
numbers → update `docs/test-suite.md` → release → ping
`#foresight-fde-validation` only if Auto-ahoy must pin a new release or if
failure rates will change materially.

## Done checklist (new SKU)

- [ ] Model arm in `nvidia_models.sh` or `amd_models.sh` with real floors
- [ ] AMD: vendored `rvs_level_4.conf` (and level 5 if applicable)
- [ ] Calibration comment cites date, host(s), min-best, headroom
- [ ] `README.md` + `docs/test-suite.md` updated
- [ ] Release published to GitHub Releases / ghcr
- [ ] `#foresight-fde-validation` notified with `--gpu-model` + release tag
- [ ] Auto-ahoy toggle confirmed enabled for the target fleets
