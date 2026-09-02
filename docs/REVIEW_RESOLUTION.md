# Review resolution addendum

- PR: #1
- Base: `main`
- Resolution scope: the five existing review threads listed below.
- This document is a normative design addendum. It records the accepted resolution contract; it does not claim that implementation or tests have already run.
- Bot review is not retriggered.

## PRRT_kwDOTN39hM6OZ6Zg — production CSP baseline

Problem: a CSP that lists only selected fetch directives does not establish a default-deny privacy boundary.

Resolution: the production policy MUST define an explicit baseline including `default-src 'self'`, `script-src 'self'`, `style-src 'self'` (with only documented production-safe exceptions), `worker-src 'self' blob:` when workers are required, `img-src 'self' blob: data:` only when required, `media-src 'self' blob:` only when required, `connect-src 'none'` when the product has no network dependency, `form-action 'none'`, `base-uri 'none'`, and `object-src 'none'`. Any exception MUST correspond to a documented runtime dependency. Development HMR policy is separate and MUST NOT be used as the production claim.

Focused verification before resolving this thread: inspect the deployed production response/meta policy and exercise all declared resource types; assert no undeclared external connection or form submission is possible.

## PRRT_kwDOTN39hM6OZ6Zh — continue playback volume

Problem: continuing after an intro/fade can leave the audio volume at zero.

Resolution: the continue path MUST cancel the fade, restore the intended volume (normally 1.0 or the user’s saved level), and only then call `play()`. The order MUST be deterministic and a rejected play promise MUST leave an observable recoverable state.

Focused verification before resolving this thread: complete the intro, invoke continue, and assert the volume before/at the play call and after playback starts.

## PRRT_kwDOTN39hM6OZ6Zj — four-choice distinctness

Problem: counting raw tracks does not prove that four answer choices are distinct after normalization.

Resolution: four-choice mode MUST construct a set of at least four distinct normalized `titleKey` values. If fewer than four exist, it MUST use a documented fallback or disable the mode; raw record count is insufficient.

Focused verification before resolving this thread: provide duplicate titles with different casing/spacing and assert that the choice gate uses normalized distinct keys.

## PRRT_kwDOTN39hM6OZ6Zk — Vite/PWA base path

Problem: deployment under `/musicbattle/` can break assets, manifest, and service-worker scope when paths assume root.

Resolution: production build configuration MUST set Vite `base: '/musicbattle/'`. PWA `scope`, `start_url`, and registration/fetch paths MUST remain under that prefix.

Focused verification before resolving this thread: build and inspect emitted asset URLs, manifest, and service-worker scope for the exact prefix.

## PRRT_kwDOTN39hM6OZ6Zl — playback capability versus metadata

Problem: metadata parsing cannot prove that a browser can decode, play, or access a DRM-protected track.

Resolution: metadata parsing MUST only classify tags and basic file information. A playback probe and/or actual playback error path MUST own the unplayable/DRM/decoder classification and MUST select a documented replacement or user-visible failure state. A parse success or failure MUST NOT by itself claim DRM support or absence.

Focused verification before resolving this thread: exercise a valid playable asset, a decoder failure, and a DRM/restricted asset (where available), and assert the playback-owned classification.

## Verification status

The checks above are required acceptance criteria for implementation. This addendum intentionally reports no test result and no implementation-complete status.