# TILED_API_KEY Propagation Investigation

## Context
CMS profile failed with Tiled authentication: `from_profile("nsls2", username=None)` was stripping the Authorization header we set in the nsls2.yml profile, resulting in anonymous access denial.

## Three Possibilities

We haven't yet confirmed which layer is the issue. Each has a different owner:

### 1. CI Harness Issue (Most Likely)
**Description:** `TILED_API_KEY` isn't actually propagating to the profile-test step for cms.

**How it should work:** The "Start Tiled and create profiles" step exports `TILED_API_KEY` via `echo "TILED_API_KEY=..." >> $GITHUB_ENV`. This should make it available to all subsequent steps in the job.

**Evidence:** We have seen TILED_API_KEY propagate correctly for at least one beamline earlier in the session (during chx debugging), so the mechanism *can* work. But we haven't directly verified it for cms in the current run.

**Debug:** Added `echo "DEBUG: TILED_API_KEY_LEN=${#TILED_API_KEY}"` to "Run profile startup tests" step. If output shows `TILED_API_KEY_LEN=6`, the env var is present. If `TILED_API_KEY_LEN=0`, env propagation is failing.

### 2. Profile Issue (Also Likely)
**Description:** CMS profile calls `from_profile("nsls2", username=None)` without explicitly passing `api_key=` kwarg, relying entirely on env variable fallback.

**Why it's fragile:** The tiled client only checks the env var if `api_key` kwarg is omitted. If env is missing or unset, authentication fails. This is valid production pattern (env-based config), but it's fragile in CI where env setup is complex.

**Mitigation:** CMS could defensively pass `api_key=os.getenv("TILED_API_KEY")` to be explicit about the dependency.

### 3. Tiled Client Design Issue (Less Likely, but worth noting)
**Description:** The tiled client's property setter strips a deliberately-set `Apikey Authorization` header when `api_key=None` is passed.

**Behavior:** Even though our nsls2.yml contains `headers: Authorization: "Apikey secret"`, calling `from_profile("nsls2", username=None)` (which implicitly sets `api_key=None`) overwrites this header with empty/default auth.

**Upstream fix potential:** The tiled client could preserve explicitly-set headers or fail noisily if there's a conflict between kwarg and YAML config.

## Investigation Path

Priority order to investigate:
1. **CI propagation** — Check next run's debug output. If `TILED_API_KEY_LEN=6`, env is working → focus on profile/tiled client behavior.
2. **Profile defensive coding** — If (1) confirms env propagation, update CMS to pass `api_key=` explicitly.
3. **Tiled upstream** — If (1) and (2) don't solve it, file issue with tiled about header stripping behavior.

## Next Steps
1. Run HEX test to see if similar auth issue occurs (rules in/out CMS-specific profile bug)
2. If CMS-specific: check debug output to see if TILED_API_KEY present
3. If TILED_API_KEY absent: investigate env propagation mechanism in CI
4. If TILED_API_KEY present but auth still fails: update CMS to pass api_key= explicitly
