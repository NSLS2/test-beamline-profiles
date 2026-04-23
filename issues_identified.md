# Issues Identified During CI Beamline Profile Testing

This file tracks beamlines where CI uncovered profile-collection-side issues.
When details are incomplete, entries are marked as provisional until logs are reviewed again.

## Confirmed Profile Collection Issues

### AMX

- Status: Confirmed profile issue (not CI harness).
- Symptom: Startup failure in startup/28-governor.py.
- Error signature: AttributeError related to gov.gov.Robot.
- Owner: amx-profile-collection maintainers.

### BMM

- Status: Confirmed profile issue (not CI harness).
- Symptom: Startup failed in startup/00-populate-namespace.py.
- Error signature: ConnectionError connecting to xf06bm-ioc2:6379 (temporary name resolution failure).
- Cause: Profile uses a legacy hardcoded Redis hostname instead of CI/deployment-configured Redis endpoint.
- Suggested fix:
  - Remove hardcoded xf06bm-ioc2 usage.
  - Resolve Redis endpoint from profile config or environment.
- Owner: bmm-profile-collection maintainers.

### CMS

- Status: Confirmed profile issue (not CI harness).
- Symptom: Startup failed in startup/00-startup.py during Tiled client creation.
- Error signature: RuntimeError indicating API key authentication required.
- Key diagnostic proof:
  - CI debug output showed DEBUG: TILED_API_KEY_LEN=6.
  - This confirms TILED_API_KEY is present in the startup-test step.
- Cause: CMS calls from_profile("nsls2", username=None) without explicit api_key argument.
- Suggested fix:
  - Pass api_key explicitly, for example:
    from_profile("nsls2", username=None, api_key=os.getenv("TILED_API_KEY"))
- Owner: cms-profile-collection maintainers.

### CSX

- Status: Confirmed profile configuration issue.
- Symptom: Redis SSL connection behavior mismatched expected CI/deployment path.
- Error signature: Redis SSL port/path mismatch in profile assumptions.
- Cause: Profile-side Redis SSL configuration mismatch.
- Owner: csx-profile-collection maintainers.

### HXN

- Status: Confirmed profile issue (dependency branch pin).
- Symptom: pip install failed while resolving hxnfly dependency from git.
- Error signature:
  - Failed to build git+https://github.com/NSLS-II-HXN/hxnfly@master
  - git checkout -q master failed
- Cause:
  - Upstream hxnfly repository does not have a master branch (main exists).
  - Profile dependency pin references @master.
- Suggested fix:
  - Update profile dependency reference from @master to @main (or another valid ref).
- Owner: hxn-profile-collection maintainers.

### HEX

- Status: Confirmed profile issue (device startup path).
- Symptom: Startup failed in startup/09-panda.py while connecting Panda device.
- Error signature:
  - ophyd_async.core._utils.NotConnectedError
  - panda: NotConnectedError: pva://XF:27ID1-ES{PANDA:1}:PVI
- Cause:
  - Profile attempts live PVA Panda connection during CI startup instead of fully mocking/skipping this hardware path for CI.
- Suggested fix:
  - Gate Panda connection logic for CI mode and use mock path in CI.
  - Ensure RUNNING_IN_NSLS2_CI (or equivalent) correctly bypasses real PVA device connection in startup.
- Owner: hex-profile-collection maintainers.

### ISS

- Status: Confirmed profile issue (upstream dependency).
- Symptom: Startup failed during Python compilation of isstools package.
- Error signature:
  - SyntaxError: invalid syntax in /usr/share/miniconda/lib/python3.13/site-packages/isstools/batch/batch.py line 19
  - `if tables[name][[column]].isnull().get()s.all():` (note the `.get()s` typo)
- Cause:
  - isstools package has a syntax error (typo: `.get()s` should be `.get_values()` or `.values`).
  - Likely Python 3.13 compatibility issue or undetected typo in upstream isstools.
- Suggested fix:
  - Update isstools dependency or submit PR to fix typo in batch.py line 19.
  - Change `.get()s` to `.values` or appropriate method.
- Owner: iss-profile-collection maintainers (or isstools maintainers).
### LIX

- Status: Confirmed profile issue (conda channel configuration).
- Symptom: pixi install failed with invalid conda channel.
- Error signature:
  - UnavailableInvalidChannel: HTTP 404 Not Found for channel globus-sdk
  - `The channel is not accessible or is invalid.`
- Cause:
  - Profile has `globus-sdk` configured as a conda channel in pixi.toml, but this channel is not accessible at https://conda.anaconda.org/globus-sdk.
  - Channel may be misspelled, deprecated, or requires authentication.
- Suggested fix:
  - Remove or correct the `globus-sdk` channel reference in pixi.toml.
  - If globus-sdk packages are needed, verify the correct channel URL or use conda-forge alternative.
- Owner: lix-profile-collection maintainers.

### RSOXS

- Status: Confirmed profile issue (line ending format).
- Symptom: `.ci/bl-specific.sh` execution failed with shell error.
- Error signature:
  - `.ci/bl-specific.sh: line 2: $'\r': command not found`
- Cause:
  - `.ci/bl-specific.sh` file has Windows-style line endings (CRLF) instead of Unix-style (LF).
  - When bash tries to execute the script, it encounters the carriage return character as a literal command.
- Suggested fix:
  - Convert `.ci/bl-specific.sh` line endings from CRLF to LF.
  - In git: `git checkout --theirs` followed by `git config core.safecrlf false` and `dos2unix .ci/bl-specific.sh`, or use editor to change line endings.
  - Alternatively, ensure all `.ci/*.sh` scripts use LF line endings in git attributes.
- Owner: sst-rsoxs-profile-collection maintainers (or broader SST multi-endstation issue).

## Provisional Profile Collection Issues (Need Log Refresh for Full Error Detail)

### CDI

- Status: Provisionally categorized as profile issue based on prior triage.
- Detail level: Specific traceback not currently captured in this tracker.
- Owner: cdi-profile-collection maintainers.

### CHX

- Status: Provisionally categorized as profile issue based on prior triage.
- Detail level: Specific traceback not currently captured in this tracker.
- Owner: chx-profile-collection maintainers.

## CI Harness Attribution Summary

- Current evidence indicates the above entries are profile-collection-side issues.
- For CMS specifically, CI env propagation was verified healthy by the TILED_API_KEY length diagnostic.

## CI Harness Gaps (Fixed)

### FMX

- Status: CI harness gap identified and fixed.
- Symptom: startup/01-bluesky.py failed opening:
  - /nsls2/data/fmx/shared/config/bluesky/logs/startup_log.log
- Error signature: FileNotFoundError for missing parent directory.
- Root cause: CI setup did not create /nsls2/data/${BEAMLINE_ACRONYM}/shared/config/bluesky/logs.
- Fix applied in harness:
  - Beamline directory setup now creates:
    - /nsls2/data/${BEAMLINE_ACRONYM}/shared/config/bluesky/logs
