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
