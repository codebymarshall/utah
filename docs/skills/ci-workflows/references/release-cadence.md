# Release cadence and factory pin

The cadence is RFC'd in #336. What runs today:

- Open pull requests against `main`, never `testing`. `sync-main-to-testing.yml`
  resets `testing` to `main` on every push to `main` and again nightly on its
  own `20 22 * * *` schedule, so a commit merged straight into `testing` is
  orphaned: #404 was lost this way and had to be re-landed. `build.yml` runs on
  every pull request and declares `push: branches: [testing]`, but that trigger
  is not how a `main` commit reaches the image tags: the sync pushes `testing`
  with the workflow's own `GITHUB_TOKEN`, and a `GITHUB_TOKEN` push starts no
  workflow. `sync-main-to-testing.yml`'s `build` job therefore dispatches the
  build explicitly (`gh workflow run build.yml --ref testing`) once the sync
  job returns, which is the path that actually produces the images.
- `:testing` is the latest E2E-green commit, not the branch tip, and it only
  moves forward. The tags move in `post-testing-e2e.yml`, after the LUKS ISO
  matrix and the production-ISO composition both pass, and only if the tested
  commit is the promoted commit or a descendant of it
  (`scripts/promotion-ratchet.py`; see `iso-e2e.md`). Merges to `main` while an
  E2E runs are safe and never block it: contribution is never paused to
  promote. An out-of-order older run is refused with a warning and cannot move
  `:testing` backwards. `promote-testing-to-main.yml` is the daily 04:00 UTC
  heartbeat that refreshes the squash PR from the `testing` branch; it is not
  what moves image tags.
- `:stable` moves in `execute-release.yml` on every promotion push to `main`.
  First the `e2e-evidence` job runs `promotion-ratchet.py release-gate`: for
  each flavor it takes the digest `:testing` points at, recovers the commit it
  was built from, and requires the `e2e/post-testing/<image>` success status
  that `post-testing-e2e.yml` recorded for exactly that digest on exactly that
  commit. No such evidence, no release; the testing branch tip is never
  consulted, so landing commits during a release can neither block nor bless
  it. The release then retags from the commit-addressed
  `testing-<yyyymmdd>-<sha7>` tag that gate verified (not `:testing` again) and
  runs `run_release_gate: true` over `smoke,common` against the exact
  candidate digests, with no scenarios excluded. `gate_test_ref` pins the
  gate to the testsuite commit `post-testing-e2e.yml` validated (`test_ref`,
  #773); bump both together, a test enforces it. If a scenario group goes red
  for a reason outside this repo, scope it with `gate_extra_tags`
  (actions#655) plus the matching `extra-tags` in `post-testing-e2e.yml` and a
  tracking issue; never switch the whole gate off. A weekly promotion rather
  than a per-promotion one is still an open question in #336 -- a maintainer
  policy call, not a code gap; nothing in this tree should encode a guess.
  The promotion PR's own gate (`promote-testing-to-main.yml`) keeps
  `run_e2e: false` on purpose: the pinned `reusable-promote-squash.yml` keys the
  E2E status to the `testing` branch tip, the tip-equality assumption this
  design removes. Turning it on needs the reusable workflow to accept the
  promoted commit as `head_sha` (projectbluefin/actions); until then the
  E2E requirement is enforced at release time by `e2e-evidence`.

`ARG PACKAGE_IMAGE_SHA` is the digest of `ghcr.io/projectbluefin/utah-packages`,
the RPM repository every image installs from. The built-in Dockerfile manager
does not discover this ARG indirection, so Utah's local regex manager reads
both that pin and `packages/utah-packages.repo`'s `# factory-pin:` stamp as
occurrences of one Docker dependency tracking `latest`.

Digest updates are grouped into one Renovate PR. Both occurrences must change
to the same digest: the stamp invalidates the package transaction's layer
cache, while the ARG selects the repository and labels its provenance. The
existing equality and full transaction checks remain the adoption gates.

The extraction and real replacement were exercised with Renovate 44.132.2;
the inherited local preset is `config:recommended`. This replaces the separate
scheduled factory updater rather than running two bots over the same pins.
No `image-versions.yml` alias or second literal pin is introduced. Renovate's
PR event runs normal CI; every update still requires independent review.
