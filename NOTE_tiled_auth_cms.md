# Note: Tiled auth failure in cms-profile-collection (and similar profiles)

**Status:** Not fixable in this harness — needs a profile-collection PR or a tiled-client change.
**Observed:** 2026-04-23 on `cms` in the `update-to-use-pixi-env` CI run.
**Affects:** cms, likely any profile calling `from_profile("nsls2", username=None)` without an `api_key` kwarg.

## Symptom

The profile's startup fails at:

```python
# cms-profile-collection/startup/00-startup.py, line 63
tiled_reading_client = cat = from_profile("nsls2", username=None)["cms"]["raw"]
```

with:

```
RuntimeError: This server requires API key authentication.
```

Preceded by the print `"Initializing Tiled reading client...\nMake sure you check for duo push."` — indicating the profile expects interactive authentication (Duo 2FA) in production, which is impossible headless in CI.

## Root cause

There is a subtle interaction between the profile YAML, tiled's client, and the `TILED_API_KEY` env var that makes it impossible for the harness alone to provide authentication to this call.

1. Our CI writes a `nsls2` tiled profile at `~/.config/tiled/profiles/nsls2.yml` with:
   ```yaml
   nsls2:
     uri: https://tiled.nsls2.bnl.gov
     headers:
       Authorization: "Apikey <key>"
   ```
2. cms calls `from_profile("nsls2", username=None)` → `from_uri(uri=..., headers={Authorization: "Apikey <key>"}, username=None, api_key=None)`.
3. Inside `tiled.client.context.Context.__init__`:
   - `api_key = os.getenv("TILED_API_KEY")` — falls back to env if not passed as kwarg.
   - `client.headers = headers` sets the `Authorization: "Apikey <key>"` header from the profile.
   - Then `self.api_key = api_key` runs the property setter — see `tiled/client/context.py:507-513`:
     ```python
     @api_key.setter
     def api_key(self, api_key):
         if api_key is None:
             if self.http_client.headers.get("Authorization", "").startswith("Apikey "):
                 self.http_client.headers.pop("Authorization")
         else:
             self.http_client.headers["Authorization"] = f"Apikey {api_key}"
     ```
     **If `api_key` is `None` and the header starts with `Apikey `, the setter removes the header** — even though it was set deliberately from the profile.
4. So the `headers: Authorization: "Apikey ..."` trick only survives if `TILED_API_KEY` is also set in the env. The env-var path is the **only** path that works.

The harness does set `TILED_API_KEY` via `$GITHUB_ENV` in the "Start Tiled and create profiles" step, which should propagate to later steps. If this path is somehow not working for cms specifically, that's a separate CI-propagation bug to diagnose with a debug echo.

## Why this can't be fixed in the harness

- We can't make `from_profile("nsls2", username=None)` authenticate without `TILED_API_KEY` in env (by design of the tiled client). We're already setting it.
- Embedding the key in the profile YAML via `headers:` is defeated by the property setter.
- Making the local tiled server accept anonymous reads *would* work, but would mean switching from `tiled serve catalog --temp` to a config-driven setup — larger change, orthogonal to the matrix work, and arguably makes the mock less production-faithful.

## What should change (by repo)

**cms-profile-collection** (preferred): Make the tiled calls CI-friendly. Options:

1. Pass the API key explicitly (drop `username=None`, add `api_key=os.getenv("TILED_API_KEY")` fallback):
   ```python
   tiled_reading_client = cat = from_profile(
       "nsls2",
       api_key=os.getenv("TILED_API_KEY"),
   )["cms"]["raw"]
   ```
   This works in production (interactive login sets env) and in CI (harness sets env).

2. Or guard for CI / non-interactive shells:
   ```python
   if os.getenv("CI") or not os.isatty(sys.stdin.fileno()):
       # skip interactive tiled fetch, or fall back to local-only
       tiled_reading_client = cat = from_profile(
           "nsls2",
           api_key=os.getenv("TILED_API_KEY"),
       )["cms"]["raw"]
   else:
       tiled_reading_client = cat = from_profile("nsls2", username=None)["cms"]["raw"]
   ```

The first option is cleaner and unified. Local cms checkout for reference: `/home/asligar/git_projects/cms-profile-collection`.

**tiled (optional)**: Reconsider the property setter's "strip the `Apikey` header on None" behavior. If a profile sets a header explicitly, the Context constructor arguably shouldn't clobber it. That's an upstream tiled discussion.

## Related CI env verification

If upstream profile fixes don't happen quickly and the harness wants to rule out a propagation bug, add to the "Run profile startup tests" step:

```bash
echo "TILED_API_KEY_LEN=${#TILED_API_KEY}"
```

If that prints `TILED_API_KEY_LEN=0`, the env var isn't propagating and that's a harness bug. If non-zero, the env is fine and the failure is the tiled-client behavior described above.
