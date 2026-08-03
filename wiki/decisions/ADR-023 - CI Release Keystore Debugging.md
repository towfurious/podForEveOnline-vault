---
title: ADR-023 - CI Release Keystore Debugging
type: decision
tags: [adr, android, ci, keystore, gh-actions]
aliases: [Tag number over 30]
created: 2026-07-24
updated: 2026-07-24
sources: []
status: active
adr-status: Accepted
---

# ADR-023 — `android-release` CI failure: empty secret, not a keystore-format bug

## Status
Accepted 2026-07-24.

## Context
With all 4 GitHub Actions secrets in place (see [[Guide - App Store Launch Readiness]] P0 #1), the `android-release` job was triggered for the first time. `androidApp:bundleRelease` failed inside AGP's `FinalizeBundleTask`:
```
com.android.ide.common.signing.KeystoreHelper.getCertificateInfo
Caused by: java.io.IOException: Tag number over 30 is not supported
```
The same keystore built successfully locally (macOS, JDK 21.0.11 Homebrew) every time — only the CI environment (Linux, JDK 21.0.11 Temurin) failed, identically, every time.

## Decision (the actual fix)
The reconstructed keystore file in CI was **0 bytes**. `ANDROID_KEYSTORE_BASE64` was set via `gh secret set NAME < <(base64 -i androidApp/keystore/podforeve-release.jks)` run from a shell whose working directory wasn't the repo root — the relative path resolved to nothing, `base64 -i` produced no stdout, and `gh secret set` happily stored an **empty string** as the secret's value with no error. `echo "$ANDROID_KEYSTORE_BASE64" | base64 -d > file` then "succeeds" (empty in, empty out) with no error either — the workflow's own reconstruction step showed green. AGP's bundle-finalize code never checks for an empty/too-short keystore file before attempting to parse it as ASN.1, so the actual failure surfaces four steps later as a wildly misleading certificate-parsing error instead of "file is empty."

Confirmed empirically, not guessed: added a temporary CI step running `sha256sum`/`wc -c`/`keytool -list` on the reconstructed file. Output: `e3b0c442...` (the well-known SHA-256 of an empty input) and `0` bytes — conclusive. Re-set the secret with an **absolute path** and a length sanity-check before sending (`echo "${#B64}"`, expected 3008 chars for this keystore); re-ran CI; all 3 jobs green, `BUILD SUCCESSFUL`, signed AAB artifact uploaded.

**Permanent fix in `.github/workflows/ci.yml`**: replaced the temporary debug step with a small guard right after keystore reconstruction — fails fast with `::error::` if the decoded file is under 100 bytes, instead of letting a corrupt/empty secret surface as an unrelated AGP stack trace four steps later.

## A wrong turn along the way (kept here so it isn't repeated)
Before finding the real cause, the keystore's format (`Keystore type: PKCS12` — the JDK 9+ `keytool` default) looked like a very plausible explanation: "Tag number over 30" is a real, documented class of bug where AGP's old embedded ASN.1 parser chokes on how modern JDKs encode PKCS12 structures, and multiple GitHub issues describe exactly this symptom. Regenerated the keystore with `-storetype JKS` (same password/alias/DN, so only 1 of 4 GitHub secrets needed updating) on that theory — **it didn't fix anything**, because the actual keystore content was never the problem; the file CI was reading was empty regardless of which one existed locally. The JKS-format keystore is being kept anyway (zero downside pre-launch, and it sidesteps a whole real class of PKCS12-parsing fragility even though it wasn't the cause here) — but the lesson is: an empty/corrupt file was always going to produce *some* confusing downstream parser error, and the specific error text ("Tag number over 30") is not reliable evidence of *which* upstream cause is at fault. Verify the actual bytes before chasing a format theory.

## Consequences

**Positive**
- `android-release` CI job is now genuinely proven end-to-end, not just YAML-valid: real signed AAB produced and uploaded as an artifact from a fresh CI runner using only repo secrets.
- The new guard step turns any future "secret went in empty/truncated" mistake into an immediate, clear, one-line error instead of a multi-step misdirection through AGP internals.
- Release keystore is now `JKS` format, avoiding an entire class of PKCS12/modern-JDK encoding quirks documented across multiple unrelated tools (Android, IntelliJ/YouTrack IDEA-304481) — a reasonable belt-and-suspenders choice even though it wasn't this incident's cause.

**Negative / watch items**
- `gh secret set NAME < <(command)` (or any pattern where a failed/empty command silently feeds empty stdin) has no built-in safety net — GitHub accepts and stores empty secret values without complaint. Always sanity-check length/content *before* sending, not just check the `gh` command's own exit code.
- The keystore was regenerated once already (2026-07-23 → 2026-08-02) purely on a wrong hypothesis. Harmless pre-launch (never submitted to Play Store), but worth remembering that changing the release signing key post-launch would NOT be this casual — Play Store enforces the same signing key for updates outside its key-rotation program.

## Alternatives considered
- **Reverting to the original PKCS12 keystore** now that the real cause is known — rejected: no evidence PKCS12 would have failed once the secret was correctly non-empty, but no evidence it wouldn't hit some other JDK-version-specific parsing edge case either; JKS is already working and proven, reverting would be pure unforced churn.
- **Leaving the debug step in place instead of a permanent guard** — rejected: `sha256sum`/`keytool -list -v` output on every run is noisy and unnecessary once the failure mode is understood; a tight, purpose-built size check is more maintainable.

## References
- [[Guide - App Store Launch Readiness]] — P0 #1 / P1 #7.
- `.github/workflows/ci.yml` — `android-release` job.
