# GUI acceptance contract

The bridge executes a visual candidate. The independent validator is the only
component allowed to turn execution output into an accepted GUI result.
`status=passed` in `result.json` is therefore necessary but not sufficient.

## Required trust chain

1. The queue job names one active task id, one exact Developer commit SHA, and
   one published executable/PCK fingerprint.
2. `/Users/Shared/godot-playtest/runtime/submit-card-playtest.sh` requires a
   safe `--task-id` and hexadecimal `--candidate-sha`. It creates the job id and
   derives the one valid result directory from that id.
3. The bridge runs as `godot-test`. The bridge UID and the launched app UID must
   equal the independently resolved `godot-test` UID.
4. The tested executable and PCK hashes must equal the immutable build manifest
   and the queue job's expected fingerprints. The manifest must also contain a
   full `source_sha`, certify `source_clean=true`, and match the candidate SHA.
   Builds from a dirty worktree are rejected.
5. Every accepted interaction must be delivered through a call scoped to the
   verified `pid + window_id`. The runner accepts only an exact-window receipt,
   an AX receipt for a background call on that window, or a process-bound
   synthetic/foreground receipt whose summary names the verified PID. Desktop
   input, PID-only pointer helpers, Terminal fallback, an unscoped global key or
   click, and `viewport.push_input` are not acceptance routes.
6. Interactions obtained from Godot card/control rectangles are recorded as
   `godot_control_rect`. AX controls use `accessibility_control`; focused
   keyboard actions use `focused_control`. Deck filters/cards, detail close,
   battle end-turn, and confirmation dialogs use native AX controls. Pixel
   input is reserved for live Godot control rectangles and custom-drawn
   surfaces that do not expose an actionable AX element. Hardcoded
   reference-layout points are rejected in a passed result.
7. Each visual checkpoint produces an exact pair:
   `raw/<checkpoint>.png` and `<checkpoint>-native.png`. The two filename sets
   must be one-to-one; an unmatched render or native capture rejects the run.
   At least two checkpoints are required, and duplicate render or native hashes
   cannot masquerade as separate visual states.
   Native capture metadata binds file name, SHA-256, bytes, and independently
   decoded pixel dimensions.
8. `raw-index.json` binds every application render by name, hash, byte count,
   width, and height. Sealing changes `raw/` to mode `0555` and every render to
   `0444`, so the main-account validator can read evidence produced by
   `godot-test` without granting write access. A passed result must contain real
   render/native evidence and an evidence-backed observation assertion.
9. A passed result requires these ordered executable-scope cleanup records:
   `bridge-before-launch`, `cards-before-launch`, `cards-exit`, and
   `bridge-post-job`, all with `verified_zero=true`, no remaining PIDs, and the
   expected GUI UID.
10. The bridge writes `processed/<job-id>.completed.json` only after launch,
    post-job cleanup, normalization, and sealing have reached a terminal state.
    `submit-card-playtest.sh` does not return a failure or success before this
    marker matches the current `result.json` hash.
11. The sealer hashes every top-level evidence file. The validator rejects a
    missing, added, changed, or under-declared top-level file.
12. Mandatory capability evidence must name the same job id and candidate SHA.
13. The result must name the exact task-selected GUI scenario (`demo`). The
    state-machine acceptance path supplies the expected scenario independently,
    so a valid run of the wrong fixture cannot be accepted for a task.

## Validator command

```bash
automation/godot/validate-card-playtest-result.sh \
  --result-dir /Users/Shared/godot-playtest/results/cards-... \
  --task-id deck-upgrade-ui \
  --developer-commit 40-character-sha \
  --expected-demo deck \
  --ticket .game-auto/capability-ticket.json
```

`automation/nightshift/longrun-control.sh record-test passed` and
`automation/nightshift/night-control.sh record-test passed` call this validator
before changing state. A test directory in the `passed` state is not enough.

## Native UI contract

Semantic AX controls are the preferred route for every actionable surface.
Custom-drawn controls should expose semantic state that names the affected card,
control, or action. Existing card, reward-option, and reward-confirm rectangles
come from live Godot `Control` nodes and are the fallback only where an
actionable AX element is unavailable. The runner keeps one exact-window pointer
helper and one AX helper; duplicate desktop, isolated-pointer, double-click, and
reference-coordinate paths are not part of the native runner. Because the
sealed evidence is local, the validator assumes that the runner did not
intentionally forge its own action ledger; it detects accidental drift and
post-run tampering, not an adversarial process running with the same UID.

## Process cleanup contract

The existing exact-scope reaper is the model for the rest of the system. A valid
result requires these `verified_zero=true` records:

- `bridge-before-launch`
- `cards-before-launch`
- `cards-exit`
- `bridge-post-job`

On a passed run, these records must be present in that order and the final
bridge record must prove zero surviving processes. Failed execution attempts
still publish `result.json`, cleanup diagnostics, and a completion marker, but
they are not sealed or eligible for acceptance.

## Seal boundary

The seal is local tamper evidence: owner and mode checks, complete file hashes,
result-to-manifest binding, and independent image decoding. It is not a
cryptographic signature and must not be described as one. Stronger protection
against a same-UID attacker requires a privileged signer and is outside this
contract.

## Negative tests

`validate-card-playtest-result.test.sh` verifies rejection of stale candidates,
builds whose source commit differs from the candidate, dirty-source builds,
wrong owners, a missing or wrong GUI scenario, test-only input, failed cleanup,
missing native captures, unmatched render/native checkpoints,
hardcoded-reference interaction, missing geometry provenance, post-seal
tampering, added files, missing bridge completion, and capability evidence that
is not linked to the tested run. It also proves both state-machine entry points
reject an unsealed result.
