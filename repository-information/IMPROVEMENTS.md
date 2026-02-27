# Potential Improvements

Ideas and optimizations to explore — no commitment, investigated when time allows.

## GAS Auto-Update: Switch from CacheService to PropertiesService

**Current approach:** When a new GAS version is deployed, `pullAndDeployFromGitHub()` writes the version string to `CacheService.getScriptCache()` with key `"pushed_version"`. Client-side JS polls `readPushedVersionFromCache()` every 15 seconds. If the version differs, it triggers the "Code Ready" blue splash reload.

**Problem:** CacheService entries expire after 1 hour (max 6 hours). If a client doesn't poll within that window — e.g. a tab left open overnight — it never learns about the update. The cache can also vanish if Google recycles the GAS runtime before the TTL expires.

**Proposed improvement:** Replace `CacheService.getScriptCache()` with `PropertiesService.getScriptProperties()` for storing the deployed version:
- **Persists indefinitely** — survives runtime recycling, no TTL expiration
- **Same broadcast semantics** — script-level properties are shared across all clients
- **No cleanup needed** — overwrite the same key each deploy
- **Negligible performance difference** — slightly slower than CacheService, but irrelevant at a 15-second polling interval

**Changes required:**
1. In the deploy function: replace `cache.put("pushed_version", version, 3600)` with `PropertiesService.getScriptProperties().setProperty("pushed_version", version)`
2. In `readPushedVersionFromCache()`: replace `CacheService.getScriptCache().get("pushed_version")` with `PropertiesService.getScriptProperties().getProperty("pushed_version")`
3. Rename the function to `readPushedVersion()` (dropping "FromCache") for accuracy
4. Update CLAUDE.md GAS Code Constraints section to reflect the new storage mechanism

Developed by: ShadowAISolutions
