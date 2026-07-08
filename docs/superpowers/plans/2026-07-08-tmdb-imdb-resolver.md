# TMDB IMDb ID Resolver Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** When GuessIt extracts a title/season/episode/year from a filename, resolve it to a precise IMDb ID via TMDB before falling back to OpenSubtitles' own (ambiguity-prone) name search.

**Architecture:** Add a TMDB API key field to `vlsubcom.lua`'s existing Settings dialog (persisted automatically through the file's existing generic `openSub.option` → JSON config-file save/load — no new file). Add a `resolveImdbIdViaTMDB(title, year, season, episode)` function using the file's existing `Curl.new()` HTTPS client class. Call it once, at the end of `getMovieInfo()`, only when `openSub.movie.imdbId` is still empty (i.e. the user hasn't manually entered one) — on success it fills `openSub.movie.imdbId`, which the extension's already-existing logic then uses to search by ID exclusively, skipping name search entirely. Any failure (no key, network error, no TMDB match, no `imdb_id` in the response) leaves `imdbId` empty and every existing behavior is unchanged.

**Tech Stack:** Lua (VLC 3.0 extension), `Curl.new()` (this file's own HTTPS client wrapper, `vlsubcom.lua:1502` for a working call example), `json` global (`dkjson`, already loaded at `vlsubcom.lua:609`), TMDB v3 REST API (`api_key` query-param auth).

## Global Constraints

- Personal fork, no upstream PR planned — matching the file's existing style (this is still worth doing for readability, but there is no external reviewer to satisfy).
- Blank TMDB key = feature fully inert; zero behavior change from current code.
- No new config file — reuse `openSub.option` (auto-persisted by the existing `save_config()`/`load_config()`, which generically JSON-encodes/decodes the whole table, `vlsubcom.lua:3021-3049` and `:2788-2817`).
- No new HTTP mechanism — reuse `Curl.new()` (already used for HTTPS calls elsewhere in this file, e.g. `https://api.myip.com` at `vlsubcom.lua:1499-1507`), not `vlc.stream()`.
- No new JSON library — reuse the existing global `json` (`json.decode(body, 1, true)`, matching the call shape used throughout, e.g. `vlsubcom.lua:1526`).
- URL-encode query values with `vlc.strings.encode_uri_component` (already used identically for the GuessIt API's `filename=` param at `vlsubcom.lua:4141`).
- No test framework added — this 9002-line file has no existing tests of any kind; this plan follows that convention and relies on manual verification (Task 3), not a new one.
- Every failure mode (no key, network error, no match, no `imdb_id` field) must be silent — no dialog error, only VLC's existing `vlc.msg.dbg`/`vlc.msg.err` logging convention.
- Do NOT touch the actual search-execution code (`SearchSubtitles*` functions) — the injection point is exclusively populating `openSub.movie.imdbId` before the existing "if imdbId set, search by ID" logic runs.

---

### Task 1: TMDB API key field in Settings dialog + persistence

**Files:**
- Modify: `vlsubcom.lua:1815-1894` (`interface_config()`)
- Modify: `vlsubcom.lua:2861-2886` (`apply_config()`)

**Interfaces:**
- Consumes: nothing from other tasks (first task).
- Produces: `openSub.option.tmdb_api_key` (string, `""` when unset). Persisted automatically — no new save/load code needed (see Global Constraints). Task 2 reads this field via `openSub.option.tmdb_api_key`.

- [ ] **Step 1: Add the new dialog row, shifting the message/button rows down by one**

In `interface_config()`, the current layout uses row 9 for the working-directory field, row 10 for the status message, and row 11 for the four action buttons (Save/Help/Check Updates/Close). Insert a new row 10 for the TMDB key, and shift the message to row 11 and the buttons to row 12.

Find this block (currently lines 1874-1894):
```lua
  -- Row 10: Status message (moved up from row 11)
  input_table['message'] = nil
  input_table['message'] = dlg:add_label('', 1, 10, 4, 1)

  -- Row 11: Action buttons (moved up from row 12)
  dlg:add_button(
    "💾 " .. lang["int_save"],
    apply_config, 1, 11, 1, 1)

  dlg:add_button(
    "❓ " .. lang["int_help"],
    function() show_help("config") end,
    2, 11, 1, 1)

  dlg:add_button(
    "🔄 Check Updates",
    function() check_for_updates(true) end, 3, 11, 1, 1)

  dlg:add_button(
    "❌ " .. lang["int_close"],
    show_main, 4, 11, 1, 1)
```

Replace it with:
```lua
  -- Row 10: TMDB API key (optional — used to resolve a precise IMDb ID
  -- via TMDB before falling back to OpenSubtitles' own name search)
  dlg:add_label("TMDB API key (optional):", 1, 10, 2, 1)
  input_table['tmdb_api_key'] = dlg:add_text_input(
    type(openSub.option.tmdb_api_key) == "string"
    and openSub.option.tmdb_api_key or "", 3, 10, 2, 1)

  -- Row 11: Status message (was row 10)
  input_table['message'] = nil
  input_table['message'] = dlg:add_label('', 1, 11, 4, 1)

  -- Row 12: Action buttons (was row 11)
  dlg:add_button(
    "💾 " .. lang["int_save"],
    apply_config, 1, 12, 1, 1)

  dlg:add_button(
    "❓ " .. lang["int_help"],
    function() show_help("config") end,
    2, 12, 1, 1)

  dlg:add_button(
    "🔄 Check Updates",
    function() check_for_updates(true) end, 3, 12, 1, 1)

  dlg:add_button(
    "❌ " .. lang["int_close"],
    show_main, 4, 12, 1, 1)
```

- [ ] **Step 2: Save the field's value in `apply_config()`**

Find this block (currently lines 2879-2885):
```lua
  -- Get username and password, trim whitespace
  local username = trim(input_table['os_username']:get_text() or "")
  local password = trim(input_table['os_password']:get_text() or "")

  -- Set the trimmed values
  openSub.option.os_username = username
  openSub.option.os_password = password
```

Replace it with:
```lua
  -- Get username and password, trim whitespace
  local username = trim(input_table['os_username']:get_text() or "")
  local password = trim(input_table['os_password']:get_text() or "")
  local tmdb_api_key = trim(input_table['tmdb_api_key']:get_text() or "")

  -- Set the trimmed values
  openSub.option.os_username = username
  openSub.option.os_password = password
  openSub.option.tmdb_api_key = tmdb_api_key
```

- [ ] **Step 3: Manual verification (no unit test — this is VLC dialog/config glue with no existing test convention in this file)**

Run: copy `vlsubcom.lua` to `%APPDATA%\vlc\lua\extensions\vlsubcom.lua`, restart VLC, open **View > VLSub OpenSubtitles.com > Config**.
Expected: a "TMDB API key (optional)" field appears between the working-directory field and the status message, empty by default. Type a value (any placeholder text is fine for this check), click Save, close the extension, reopen Config.
Expected: the value you typed is still there — proves it round-trips through `save_config()`/`load_config()` with no new persistence code.

- [ ] **Step 4: Commit**

```bash
git add vlsubcom.lua
git commit -m "feat: add optional TMDB API key field to Settings dialog"
```

---

### Task 2: `resolveImdbIdViaTMDB()` + injection into `getMovieInfo()`

**Files:**
- Modify: `vlsubcom.lua` (add the new function immediately before `getMovieInfo`, currently at line 3538)
- Modify: `vlsubcom.lua:3642` (inject the call at the end of `getMovieInfo`, before its `collectgarbage()`)

**Interfaces:**
- Consumes: `openSub.option.tmdb_api_key` (Task 1). `Curl.new()`, the global `json`, and `vlc.strings.encode_uri_component` (all pre-existing in this file, not built by this plan).
- Produces: `resolveImdbIdViaTMDB(title, year, season, episode) -> string | nil` — an IMDb ID like `"tt19815566"` (no `tt` stripped — see note in Step 1) on success, `nil` on any failure. Called from `getMovieInfo()`, which is the only caller in this plan.

- [ ] **Step 1: Add `resolveImdbIdViaTMDB`**

Insert this new function immediately before `getMovieInfo = function()` (currently line 3538), i.e. as the line right before it:

```lua
-- Resolve a precise IMDb ID via TMDB (search + external_ids) from the
-- title/year/season/episode getMovieInfo() already extracted. Returns an
-- IMDb ID string (e.g. "tt19815566") on success, or nil on ANY failure:
-- no key configured, network error, no TMDB search match, or no imdb_id
-- in the external_ids response. Never throws — every branch returns nil
-- instead of raising, so a failure here can never break the existing
-- name-search fallback in getMovieInfo()'s caller.
function resolveImdbIdViaTMDB(title, year, season, episode)
  local key = openSub.option.tmdb_api_key
  if not key or trim(key) == "" then
    return nil
  end
  if not title or trim(title) == "" then
    return nil
  end

  local client = Curl.new()
  client:set_timeout(10)
  client:set_retries(1)

  local has_episode = season and episode
    and tostring(season) ~= "" and tostring(episode) ~= ""
  local has_year = year and tostring(year) ~= ""

  local search_url, media_kind
  if has_episode then
    media_kind = "tv"
    search_url = "https://api.themoviedb.org/3/search/tv?query="
      .. vlc.strings.encode_uri_component(title) .. "&api_key=" .. key
  elseif has_year then
    media_kind = "movie"
    search_url = "https://api.themoviedb.org/3/search/movie?query="
      .. vlc.strings.encode_uri_component(title) .. "&year=" .. tostring(year)
      .. "&api_key=" .. key
  else
    return nil
  end

  local res = client:get(search_url)
  if not res or res.status ~= 200 or not res.body or res.body == "" then
    vlc.msg.dbg("[VLSub] TMDB search failed or returned no data")
    return nil
  end

  local ok, data = pcall(json.decode, res.body, 1, true)
  if not ok or not data or not data.results or not data.results[1]
    or not data.results[1].id then
    vlc.msg.dbg("[VLSub] TMDB search returned no usable result")
    return nil
  end
  local tmdb_id = data.results[1].id

  local ext_url
  if media_kind == "tv" then
    ext_url = "https://api.themoviedb.org/3/tv/" .. tostring(tmdb_id)
      .. "/season/" .. tostring(season) .. "/episode/" .. tostring(episode)
      .. "/external_ids?api_key=" .. key
  else
    ext_url = "https://api.themoviedb.org/3/movie/" .. tostring(tmdb_id)
      .. "/external_ids?api_key=" .. key
  end

  local ext_res = client:get(ext_url)
  if not ext_res or ext_res.status ~= 200 or not ext_res.body or ext_res.body == "" then
    vlc.msg.dbg("[VLSub] TMDB external_ids lookup failed")
    return nil
  end

  local ok2, ext_data = pcall(json.decode, ext_res.body, 1, true)
  if not ok2 or not ext_data or not ext_data.imdb_id or ext_data.imdb_id == "" then
    vlc.msg.dbg("[VLSub] TMDB external_ids had no imdb_id")
    return nil
  end

  vlc.msg.dbg("[VLSub] TMDB resolved IMDb ID: " .. ext_data.imdb_id)
  return ext_data.imdb_id
end

```

- [ ] **Step 2: Inject the call at the end of `getMovieInfo()`**

Find this block (currently lines 3640-3643, the very end of `getMovieInfo`):
```lua
  end

  collectgarbage()
end,
```

Replace it with:
```lua
  end

  if (not openSub.movie.imdbId or openSub.movie.imdbId == "")
    and openSub.option.tmdb_api_key and trim(openSub.option.tmdb_api_key) ~= "" then
    local resolved = resolveImdbIdViaTMDB(
      openSub.movie.title, openSub.movie.year,
      openSub.movie.seasonNumber, openSub.movie.episodeNumber)
    if resolved then
      openSub.movie.imdbId = resolved
    end
  end

  collectgarbage()
end,
```

Note: `extractIMDBId` (the function that parses a user-typed IMDb ID field, `vlsubcom.lua:3982-4012`) strips the `tt` prefix and stores the bare numeric ID, because "the API expects numeric ID only" per its own comment. `resolveImdbIdViaTMDB` returns the ID WITH the `tt` prefix (TMDB's `external_ids` response always includes it, e.g. `"tt19815566"`). Strip it here to match the format the rest of the codebase expects — change `openSub.movie.imdbId = resolved` above to:
```lua
      openSub.movie.imdbId = resolved:gsub("^tt", "")
```

- [ ] **Step 3: Manual verification (no unit test — network-dependent VLC glue, no existing test convention in this file)**

Run: with a TMDB key saved (Task 1), copy the updated `vlsubcom.lua` to the extensions folder, restart VLC, open a file named like `From (2022) - S02E05 - Lullaby (1080p AMZN WEB-DL x265 t3nzin).mkv` (or any filename that reproduces the original ambiguity), open **View > VLSub OpenSubtitles.com**.
Expected: search results are for "FROM" (the correct show, IMDb `tt19815566`), not "Arifureta: From Commonplace to World's Strongest". Open VLC's **Tools > Messages** (verbosity 2) and confirm a `[VLSub] TMDB resolved IMDb ID: tt19815566` line appears before the search.

- [ ] **Step 4: Commit**

```bash
git add vlsubcom.lua
git commit -m "feat: resolve precise IMDb ID via TMDB before name search"
```

---

### Task 3: Manual end-to-end regression check (blank-key path)

**Files:** none (verification only).

**Interfaces:**
- Consumes: the complete feature from Tasks 1-2.
- Produces: nothing new — confirms the "blank key = fully inert" requirement from the design doc, which Task 2's own verification doesn't cover (that one used a real key).

- [ ] **Step 1: Verify blank-key behavior is unchanged from before this plan**

Run: open **View > VLSub OpenSubtitles.com > Config**, clear the TMDB API key field, click Save. Open a file with an ambiguous title (e.g. the same "From" file from Task 2). Open **Tools > Messages** (verbosity 2) before searching.
Expected: no `[VLSub] TMDB ...` log lines appear at all (the `trim(...) ~= ""` guard in the injected block short-circuits before `resolveImdbIdViaTMDB` is ever called), and search behaves exactly as it did before this plan (name-search, same old ambiguity — this is expected and correct, it's the pre-existing behavior, not a regression).

- [ ] **Step 2: Confirm no error dialogs anywhere in this plan's code**

Re-read `resolveImdbIdViaTMDB` (Task 2, Step 1) once more: every `return nil` branch is silent (no `vlc.dialog`, no `setError`), only `vlc.msg.dbg` logging. No step in this plan changes that. No commit needed for this step — it's a self-check.
