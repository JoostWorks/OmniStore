# OmniStore developer guide

This guide is for developers who need to understand OmniStore and build on it, without having written it. It covers:

- what the app does;
- how the pieces fit together at runtime;
- where everything lives in the code;
- step-by-step recipes for the most common extensions.

The user-facing feature list is in `README.md`. This file is about how it works.

---

## 1. What OmniStore is, in one paragraph

OmniStore is a single executable for Windows and Linux. It looks like an app store, but it doesn't host any apps itself: it is a **front end for package managers that already exist**, such as WinGet, Chocolatey, Scoop, pip, npm, Cargo, .NET tools, apt, dnf, pacman, zypper, Flatpak, Snap, Homebrew, GitHub Releases and AppImage.

It searches all of them at once and merges the results so one app shows as one card. You install through whichever manager you prefer, and it keeps everything updated in the background. It also adds:

- safety ratings and "trusted author" labels;
- lists of apps you can install in one go, and can even bake into a new copy of the .exe;
- operating-system updates (Windows Update, distribution upgrades);
- ten interface languages.

## 2. How it runs

```
 ┌──────────────── omnistore.exe (Go, one static binary) ───────────────┐
 │                                                                       │
 │  main.go ── starts ──► local HTTP server 127.0.0.1:47815 (server.go)  │
 │     │                     │  serves the embedded web UI  (web/*)      │
 │     │                     │  serves a JSON API           (/api/...)   │
 │     │                     │  pushes live events (SSE)    (/api/events)│
 │     │                                                                 │
 │     ├── tray icon (tray.go)          ├── update scheduler (updater.go)│
 │     ├── OS-update scheduler          └── task queue (tasks.go)        │
 │     │   (osupdates.go)                      │ runs commands:          │
 │     │                                       ▼                         │
 │     └── providers (prov_*.go) ──► winget / choco / apt / flatpak ...  │
 │                     │          (CLI, elevated through UAC/pkexec/sudo)│
 │                     └────────► online catalogs (WinGet index, PyPI,   │
 │                                Flathub API, crates.io, ...) via httpc │
 └───────────────────────────────────────────────────────────────────────┘
          ▲
          │ http://127.0.0.1:47815/?t=<token>
 ┌────────┴─────────────────────────┐
 │ Edge/Chrome in "--app" mode      │  ← the window you see (desktop.go openWindow)
 │ web/index.html + app.js          │
 └──────────────────────────────────┘
```

Key facts:

- **The UI is a web page served by the executable itself.** Nothing is installed. The window is a Chromium-family browser opened in app mode, so it has no address bar.
- **Every API call needs a session token.** The token is generated at start-up, injected into `index.html` as `window.OMNI_TOKEN`, and sent as the `X-Token` header (or `?t=` in URLs). The server also checks the `Host` header, so other websites can't call the API.
- **Only one copy runs at a time.** `session.json` in the data folder stores the port, token and PID. A second launch finds the running copy, calls `POST /api/show` on it to bring its window forward, and exits.
- **It keeps running after the window closes,** in the tray, when "run in background" is on. The schedulers keep working.
- **The same binary is also a command-line tool** (`--search`, `--install`, `--update-all`, and others). In that mode it doesn't start the server at all.

### Start-up order (`main.go`)

1. Parse the command-line flags. `--update-snapshot` is a developer tool and exits straight away.
2. Start-up work:
   1. `cfg.init()` loads settings and the policy file, and works out the data and cache folders.
   2. `readEmbeddedKit()` and `applyKitSettings()` pick up lists baked into the .exe.
   3. Open the log file.
3. Load data:
   1. `loadCatalog()` loads the curated catalog.
   2. `loadWorkspace()` loads My lists.
   3. `initState()` loads the installed/updates cache.
   4. `initTasks()` loads the activity history.
4. If this is a command-line call, run `runCLI` and exit.
5. Otherwise:
   1. `listen()` starts the server, or hands over to a copy that is already running.
   2. Start `runScheduler()` (app updates) and `osScheduler()` (system updates).
   3. Load Chocolatey popularity in the background.
6. Open the window unless started with `--background`, then run the tray loop.

## 3. Feature tour (what a user sees)

| Page (`web/app.js` view) | What it does | Main backend |
|---|---|---|
| Discover (`V.discover`) | Hero picks, the curated catalog grouped by category, online shelves ("Popular on Flathub", "Just updated in WinGet") | `discover()` in server.go |
| All apps (`V.browse`) | Browse each manager's complete catalog, with sorting and tag filters | `handleBrowse` in browse.go |
| Categories / category (`V.categories`, `V.category`) | 13 categories, each with subcategory chips (e.g. Productivity › PDF) | `handleCategory` in browse.go, `catalog.json` |
| Search (`V.search`) | Results stream in as each manager answers, grouped into Recommended, Most downloaded and All results; less-known results are hidden | `/api/search/stream` → `searchAll` |
| App page (`V.app`) | Store page: description (Markdown), gallery, safety check with evidence, editions/add-ons/forks, all sources, install/uninstall/update | `/api/app` and `/api/app/details` → `enrichApp` |
| Favourites | Starred apps | `/api/favorites` |
| My lists (`V.lists`) | Groups → lists → apps, with notes, encrypted notes and mentions; install a whole list; Save as / Open; "Save lists to a new app" | workspace.go, kit.go |
| Installed (`V.installed`) | Everything the managers report as installed, merged into one list | state.go |
| Updates (`V.updates`) | Available updates, per-app rules (auto / notify / pin), skip a version | updater.go, state.go |
| System updates (`V.system`) | Windows Update or the distribution's upgrades, firmware, restart prompts | osupdates.go |
| Activity (`V.activity`) | The task queue, with live output of every command | tasks.go |
| Package managers (`V.managers`) | Detected managers, favourite order, set up / refresh | provider.go |
| Settings (`V.settings`) | Everything in `Settings` (config.go), including the language | config.go |

## 4. Code map

All Go files are in one package, `main`, in the project root. The web UI is in `web/` and is compiled into the binary with `go:embed`.

### Core and plumbing

| File | Responsibility |
|---|---|
| `main.go` | Flags, start-up order, command-line mode, shutdown |
| `config.go` | `Settings` (user preferences), `Policy` (admin lock-down), data and cache folders, portable mode, `AppVersion` |
| `server.go` | HTTP server, token and Host checks, about 60 `/api/...` routes; `discover()`, `searchAll()`, `enrichApp()`; the icon proxy |
| `events.go` | Server-sent events hub; `broadcast(event, data)` pushes to every open window |
| `httpc.go` | `httpGet` / `httpJSON` with an on-disk cache (`cache/http/<sha1>`). If the network is down, the stale cached copy is returned |
| `exec.go` | Running commands: `runCapture`, `runStream` (with progress and a live log), elevation (`runElevated` = pkexec/sudo, `runElevatedWindows` = a UAC batch file), `findBin` |
| `platform_windows.go` / `platform_unix.go` | OS-specific details: hidden console windows, `isElevated`, refreshing PATH |
| `desktop.go` | Opening the app window, notifications, opening links externally, shortcuts |
| `tray.go`, `autostart_*.go`, `icon.go` | Tray menu, start at sign-in, the generated icon |
| `dialogs.go` | Native Save as / Open dialogs (PowerShell WinForms on Windows; zenity or kdialog on Linux) |
| `i18n.go` | Languages list; `T()` / `TT()` translate the tray menu and notifications |

### Package managers (providers)

| File | Providers |
|---|---|
| `provider.go` | The `Prov` abstraction, `register()`, detection, `Search` / `Install` / `Uninstall` / `Upgrade`, table parsing helpers |
| `prov_win.go` | winget, choco, scoop |
| `prov_lin.go` | apt, dnf, pacman, aur, zypper |
| `prov_universal.go` | flatpak (Flathub API), snap (Snap Store API), brew |
| `prov_dev.go` | pip, pipx, npm, cargo, dotnet tools |
| `prov_github.go` | GitHub Releases (`github`), AppImage (`appimage`): download, pick the right asset, install, create shortcuts |
| `websearch.go` | The `web` provider: Wikidata and DuckDuckGo lookup for apps that are in no manager; the "learned" catalog |

### Catalogs and app data

| File | Responsibility |
|---|---|
| `catalog.json` | Curated catalog: 13 categories with subcategories, and 207 hand-picked apps mapped to their ID in each manager |
| `catalog.go` | Loads the catalog; `mergePackages` (many `Pkg` → one `App`); `finishApp` (name, icons, versions, trust); ranking |
| `winget_index.go` | The full WinGet catalog (~15.7k apps) from the winget-pkgs-index project, plus the built-in snapshot `winget-snapshot.json.gz` and the manifest reader for store pages |
| `catalogs.go` | Whole-catalog indexes for Scoop (bucket archives), PyPI (top 15k), npm, crates.io, NuGet |
| `browse.go` | All apps page, category pages with subcategories, package icons, "find other sources", install scripts |
| `trust.go` | Popularity, labels (Open source, Paid, Archived…), the 0–100 safety score with an evidence list, look-alike detection |
| `authors.go` | Trusted authors (Microsoft, Google, Mozilla…) and how each one is confirmed |
| `family.go` | Groups editions, add-ons and forks under the main app |
| `media.go` | Screenshots from READMEs and websites (the gallery) |
| `logos.go` | Real app logos from Wikidata, website icons, the README or the GitHub avatar; `/api/logo` |
| `versions.go` | Version comparison |

### State, tasks and features

| File | Responsibility |
|---|---|
| `state.go` | Cache of installed packages and updates per manager; `managerStatus` |
| `tasks.go` | Task queue: one lock per manager, live log and progress, password prompt, history in `activity.json` |
| `updater.go` | App update scheduler and policies (auto / notify / off, per-app rules, maintenance window), bundles (export/import), self-update |
| `osupdates.go` | Windows Update through the `Microsoft.Update.Session` COM API (PowerShell); Linux full-upgrade, fwupd, release upgrade, reboot detection |
| `workspace.go` | My lists: groups, lists, notes, mentions; secured notes (AES-256-GCM with PBKDF2); where the lists file lives (Save as / Open) |
| `kit.go` | "Save lists to a new app": appends the lists to a copy of the .exe and reads them back at start-up |

### Web UI (`web/`)

| File | Responsibility |
|---|---|
| `index.html` | Shell: sidebar navigation and the `#view` container |
| `app.js` | The whole UI (no framework): state `S`, router, a `V.<page>` function per page, delegated click handlers, modals, SSE listener |
| `app.css` | All styling; colours are CSS variables, with light and dark themes |
| `md.js` | Safe Markdown/HTML renderer for descriptions and notes (allow-list sanitiser) |
| `i18n.js` | Translates the page as it renders, using `i18n/<lang>.json` |
| `i18n/*.json` | Translations: `nl` is complete; `es`, `de`, `fr`, `it`, `pt-BR`, `ru`, `zh-CN`, `ja`, `ko` are basic |
| `fonts/` | Big Shoulders Display and Instrument Sans (OFL licence) |

### Tests

`*_test.go`: parsers (`parse_test`), WinGet index and catalogs (`winget_test`), kits and secured notes (`kit_test`), trust and authors, families, logos, OS-update parsing, lists location, Go-side translation. Run them with `go test ./...`. Tests that need online data use cached fixtures; when a fixture is missing, the test is skipped rather than failed.

## 5. The core data model

Everything revolves around three types.

### `Pkg`: one package in one manager (`provider.go`)

```go
Pkg{Manager: "winget", ID: "Mozilla.Firefox", Name, Version, InstalledVersion,
    Summary, Description, Publisher, Homepage, License, Icon, Screenshots,
    Tags, Downloads, Installed, Update, Extra map[string]string}
```

`Extra` is a free-form bag for manager-specific facts, for example `verified`, `store page`, `github`, `sha256`, `fork of`, `approved` or `release notes`. The trust engine and the UI read these keys.

### `Prov`: a package manager (`provider.go`)

A provider is a struct of **function fields**, registered at start-up with `register(&Prov{...})` in an `init()`. You only fill in what the manager supports:

| Field | Purpose |
|---|---|
| `ProviderInfo` | ID, name, kind (`app` / `system` / `universal` / `dev`), OS list, colour, `NeedsAdmin`, `Versions`, `Scopes`, `OnlineOnly` |
| `bins`, `versionArgs`, `versionRe` | How to find the executable and read its version |
| `search`, `details`, `installed`, `updates`, `versions`, `popular` | Queries; each returns `[]Pkg` |
| `installCmd`, `uninstallCmd`, `upgradeCmd`, `upgradeAll`, `refreshCmd`, `bootstrap` | Return `[]CmdSpec` (the command lines to run). `bootstrap` installs the manager itself |
| `customInstall`, `customUninstall` | For managers that aren't a command-line program (GitHub Releases, AppImage, web) |
| `onlineSearch` | Search works through an online API even when the program isn't installed |
| `okCodes`, `argvPrefix`, `isSupported` | Exit codes that also mean success, extra arguments (e.g. `-m pip`), extra conditions |

`CmdSpec{Args, Admin, Env, Dir, IgnoreError}`: when `Admin` is true, the command runs elevated, through UAC on Windows or pkexec/sudo on Linux.

### `App`: what the user sees as one app (`catalog.go`)

One `App` is many `Pkg`s merged together (`Sources`), plus everything worked out from them:

- name, icons, latest version, installed state;
- `Preferred` (the manager that will be used);
- `Popularity`, `Labels`, `Safety`;
- `Family` / `FamilyOf` (editions, add-ons, forks);
- `Gallery`.

`App.Key` is the stable identity used in URLs (`#/app/<key>`), favourites and lists.

### How packages become apps (`mergePackages`, `catalog.go`)

1. **Curated aliases first.** `catalog.json` maps each curated app to its IDs, for example `{"winget": "Mozilla.Firefox", "choco": "firefox", "flatpak": "org.mozilla.firefox", ...}`. Any `Pkg` matching an alias joins that app.
2. **Otherwise name keys.** `candidateKeys` normalises names and IDs: it strips installer words, "MSIX", locale segments and similar. Packages that share a key are merged.
3. **Safety rules,** to avoid fake apps riding on real ones:
   - Two different packages from the *same* manager are never merged.
   - A non-curated package never joins a curated app through its name alone; the exception is local installs (`ARP\`, `MSIX\`).
   - Developer managers (pip, npm, cargo, dotnet) only merge through the curated catalog.
4. `finishApp` then picks the name, icons, version and preferred source, and calls `computeTrust`.
5. `groupFamilies` (`family.go`) folds editions, add-ons and forks under the strongest related app.

## 6. What happens when…

### …the user searches (`/api/search/stream`)

1. `searchAll(ctx, q, …)` runs each searchable provider **in parallel**. WinGet answers from the local index instantly; the others call online APIs or the CLI. Each provider has its own timeout.
2. Every time a provider answers, the results so far are merged (`mergePackages`), ranked (`rankApps`), grouped (`groupFamilies`) and **streamed** to the page as an SSE `results` event, together with per-provider status.
3. The `web` provider (Wikidata/DuckDuckGo) runs last, and only when there are few matches.
4. The page sorts results into Recommended (well known and safe), Most downloaded, All results, and a collapsed "less-known or suspicious" group.

### …the user opens an app page

1. `/api/app` returns the app at once, from the cache of recent results or the catalog.
2. `/api/app/details` → `enrichApp` then fetches, concurrently:
   - each source's details (WinGet manifest, Chocolatey OData, Flathub appstream…);
   - other managers that also carry the app (`findOtherSources`);
   - the full trust evaluation (`computeTrust(a, true)`);
   - images and logos (`appMedia`).

   The page re-renders when the details arrive.

### …the user installs something

1. `POST /api/install {appKey, manager, id, options}`.
2. The server checks the policy (blocked apps or managers). If the chosen manager isn't installed, it builds a **setup + install** task (`newSetupInstallTask`): bootstrap the manager, then install.
3. `enqueue(task)`:
   - Tasks for the same manager run one at a time; system managers run one at a time globally.
   - The task runs the provider's `installCmd` through `runStream`.
   - Each output line is parsed for progress (`cleanLine`) and broadcast as SSE `log` / `progress` / `task` events.
   - If sudo needs a password, the task enters the `password` state and the UI asks for it. The password stays in memory only.
4. When the task finishes, the installed-state cache for that manager is refreshed and a notification is shown.

### …updates run in the background (`updater.go`)

`runScheduler` wakes on the configured interval and calls `checkAndApply`:

1. Refresh `updates` for every available manager (`state.go`).
2. `applyPolicy` decides, per update:
   - Per-app rules come first: `auto`, `notify`, `ignore`, `pin`.
   - Then the global mode: `auto`, `notify` or `off`.
   - Skipped versions, the maintenance window, `IncludeSystem`, and pause are honoured.
3. Automatic updates are queued as tasks with `Auto: true`; for the rest a notification is shown.

`osScheduler` (`osupdates.go`) does the same once a day for operating-system updates. It can also restart automatically, but only inside a configured window and with a warning.

### …lists are saved and carried

- **Saving:** `workspace.go` keeps `Workspace{Groups → Lists → Items, Notes, Mentions}`. The UI sends the whole workspace back on each change (`PUT /api/workspace`, debounced).
  - Where it's saved: `Settings.ListsFile` if the user picked one with Save as; otherwise `OmniStore lists/my-lists.json` next to the program; otherwise the data folder.
  - Secured notes are encrypted on the server. Plain text is never written to disk: the PUT handler strips it.
- **Lists inside a program file:** `kit.go` copies the running executable and appends:

  ```
  [exe bytes][gzip JSON kit][8-byte length]["OMNIKIT1"]
  ```

  At start-up, `readEmbeddedKit` checks the end of its own file for that marker. This is why "Save lists to a new app" produces a working program that carries the lists inside it.

## 7. The front end in more detail (`web/app.js`)

- **No framework or build step.** Plain JavaScript; templates are template literals. Edit the file and rebuild the Go binary (the web folder is embedded at compile time).
- **State:** a global `S` holds `info` (from `/api/info`: settings, OS, version, languages), `apps` (a Map of every app seen, by key), `tasks`, `mgr` (manager status), `ws` (My lists), `picked` (the selection) and `os` (system updates).
- **Router:**
  - `render()` reads `location.hash` (`#/app/firefox`, `#/category/productivity/pdf`) and calls `V[name](viewElement, rest)`.
  - Each page renders into a fresh `#view` element, so its event listeners don't leak into the next page.
- **Actions:** buttons carry `data-act="install"` (plus `data-key` and similar); one delegated click handler routes them with a `switch`. The My lists page has its own handler for actions whose names start with `group`, `list`, `item`, `note` or `mention`.
- **API:** `api(path, {body, method})` adds the token, sends JSON and throws on errors.
- **Live updates:** an `EventSource('/api/events')` listens for `task`, `progress`, `log`, `state`, `updater` and `os` events and redraws what is affected.
- **Icons:** each card has a list of image URLs (`a.icons`). When one fails to load, the next one is tried: store icon → real logo (`/api/logo`) → favicon → coloured letter tile. Outside images are loaded through `/api/icon?u=`, which caches them.
- **Markdown:** `renderMarkdown(text, {base, img})` in `md.js` renders it. Anything not on the allow-list is removed: no scripts or styles; images go through the proxy; links open in the normal browser.
- **Translation:** `i18n.js` watches the page with a MutationObserver and replaces English text with the translation from `i18n/<lang>.json`.
  - Keys are the exact English text, with `{}` where a value is filled in.
  - Package-sourced text sits in elements marked `translate="no"`, `.md`, `.mono` or `pre`, and is never touched.
  - So **you write the UI in English and add keys to the JSON files**. You don't need to wrap strings in a function.

## 8. Files on disk

| What | Where (Windows / Linux) |
|---|---|
| Settings, activity history, tasks | `%AppData%\OmniStore\` / `~/.config/OmniStore/` (`settings.json`, `activity.json`, `session.json`, `omnistore.log`) |
| Download and image cache | `%LocalAppData%\OmniStore\` / `~/.cache/OmniStore/` (`http/`, `icons/`, `logos.json`, `os-updates.json`, `state.json`) |
| My lists | `<program folder>\OmniStore lists\my-lists.json`, or the file chosen with Save as |
| Learned apps (found by web search) | `learned.json` in the data folder |
| Portable mode | Create a folder named `omnistore-data` (or a file named `portable`) next to the exe; everything is then stored there |
| Admin policy | `%ProgramData%\OmniStore\policy.json` / `/etc/omnistore/policy.json`, or next to the exe; see `policy.example.json` |

## 9. Recipes: extending OmniStore

### Add a package manager

1. **Write the provider.** Create `prov_<name>.go` (don't end the file name in `_windows.go` / `_linux.go`: Go treats that as a build constraint).
2. **Register it in an `init()`:**
   ```go
   func init() {
       register(&Prov{
           ProviderInfo: ProviderInfo{ID: "mypm", Name: "MyPM", Kind: "app",
               OS: []string{"windows", "linux"}, Color: "#4a90e2",
               Description: "One sentence for the Package managers page."},
           bins:        []string{"mypm"},
           versionArgs: []string{"--version"},
           search: func(ctx context.Context, p *Prov, q string, limit int) ([]Pkg, error) {
               out, err := runCapture(ctx, 30*time.Second, p.argv("search", q)...)
               if err != nil { return nil, err }
               return parseTable(out, ...), nil   // or parse JSON output
           },
           installed:    ...,   // list installed packages
           updates:      ...,   // list packages with an update
           installCmd:   func(p *Prov, r InstallReq) []CmdSpec { return []CmdSpec{{Args: p.argv("install", r.ID)}} },
           uninstallCmd: func(p *Prov, id string) []CmdSpec { return []CmdSpec{{Args: p.argv("remove", id)}} },
           upgradeCmd:   func(p *Prov, ids []string) []CmdSpec { ... },
       })
   }
   ```
   - If it has an online API, set `onlineSearch: true` and call the API with `httpJSON` (that gives you caching for free).
   - If it can install itself, add `bootstrap`.
3. **Wire it into the rest of the app:**
   - Add its ID to the default `ManagerPriority` in `config.go`.
   - Give it a display rank in `displayRank` (`catalog.go`).
   - Add its name to `managerName` (`trust.go`).
4. **Optional extras:**
   - Add its IDs to curated apps in `catalog.json` (`"i": {"mypm": "the-id"}`).
   - Teach `computeTrust` its trust signals (e.g. `Extra["verified"]`).
   - Add a whole-catalog browser in `browse.go` (`browseSources`).
5. **Test it:** add a parser test with real captured output in `parse_test.go`.

### Add or change curated apps and categories

Edit `catalog.json`:

- **An app:** `{"n": name, "c": category id, "s": short summary, "h": homepage, "i": {manager: id, ...}}`.
- **Optional app fields:**
  - `"f"`: a hero tagline;
  - `"col"`: hero colour;
  - `"icon"`: icon URL;
  - `"sc"`: subcategory IDs.
- **A subcategory:** `{"id", "name", "kw": regex on name/summary/tags, "tags": [manager tags]}`.

Then add the new category, subcategory and summary texts to `web/i18n/nl.json` (and to the other languages if you like). A remote catalog in the same format can be loaded through Settings › Curated catalog, or by placing `catalog.json` next to the exe.

### Add an API endpoint and a page

1. **Back end:** in `server.go` (or a new file with its own `xxxRoutes(mux)` called from `startServer`):
   ```go
   mux.HandleFunc("/api/thing", func(w http.ResponseWriter, r *http.Request) {
       var req struct{ ID string `json:"id"` }
       readJSON(r, &req)
       writeJSON(w, map[string]any{"ok": true})
   })
   ```
   Every `/api/` route is already protected by the token check. For live changes, call `broadcast("thing", data)` and listen with `es.addEventListener("thing", …)` in `app.js`.
2. **Front end:**
   - Add a `<a href="#/thing" data-route="thing">` link to the sidebar in `index.html`.
   - Write `V.thing = async (view, arg) => { const r = await api("/api/thing"); view.innerHTML = \`...\`; };` in `app.js`.
   - Handle its buttons with `data-act` cases in the delegated click handler.

### Add a setting

1. Add a field to `Settings` in `config.go` with a `json` tag, and a default in `defaultSettings()`.
2. Show it in `V.settings` with the helpers already there: `sw(key, title, help)` for a switch, `sel(key, title, help, options)` for a dropdown. Saving is automatic (`/api/settings` merges the patch).
3. Admins can lock it with `"locked": {"yourKey": value}` in the policy file. Locking works for any setting automatically.

### Add a trust signal or label

In `computeTrust` (`trust.go`):

- `add(kind, text, source, link, points)` adds a finding to the evidence list;
- `lab(id, text, kind, tip)` adds a chip.

`kind` is `good`, `info`, `warn` or `bad`. The grade comes from the total score and the worst finding. Count each manager once, so one manager can't inflate the score. Add the new texts to `nl.json`.

### Add a trusted author

Add one `ta(...)` line to `trustedAuthors` in `authors.go`: name, a regex for how the publisher name is written, WinGet ID prefixes, official domains, GitHub organisations and Flathub ID prefixes. The label is only given when one of those confirms the publisher. A name alone doesn't count.

### Add a language or a translation

1. Create `web/i18n/<code>.json` with `"English text": "translation"` pairs. Copy the keys from `nl.json`, which is the complete list.
   - Use `{}` for values; use `{0}` and `{1}` to change their order.
   - Plural endings like `app{}` can be ignored in the translation by writing `{0}` and leaving out `{1}`.
2. Add the language to `Languages` in `i18n.go` (`Full: true` if it covers everything).
3. **To find gaps:** switch to the language, open a page, and run `I18N._debug.untranslated()` in the browser console. It lists every visible text that has no translation yet.

### Change how OS updates work

Everything is in `osupdates.go`:

- **Windows:** the PowerShell scripts `wuSearchScript` and `wuInstallScript` use the Windows Update COM API. `classifyWU` sorts updates into groups.
- **Linux:** `linuxOSUpdates` and `installLinuxOS`.
- **The page:** `V.system` in `app.js`.

## 10. Build, run and debug

### Build

- `./build.sh` (Linux/macOS) or `./build.ps1` (Windows) does all of this:
  - runs `go mod tidy`;
  - refreshes the built-in WinGet snapshot;
  - runs the tests;
  - builds four targets into `dist/`: Windows and Linux, each amd64 and arm64.
- **Requirements:** Go 1.22 or newer. No cgo, no C compiler, no Node.js.
- **Version number:** set `AppVersion` in `config.go` and `VERSION` in the build scripts.
- **Windows builds** use `-H windowsgui` (no console window). Command-line output still works because `attachConsole` attaches to the console that started the program.

### Run while developing

```sh
go build -o omni . && ./omni --serve --no-tray     # prints the URL with the token; open it in a browser
OMNISTORE_DEV_OS=windows ./omni --serve            # on Linux: preview the Windows catalogs and pages
HOME=/tmp/test-home ./omni --serve                 # a throw-away profile
./omni --search vlc                                # command-line mode, no server
go test ./...
```

- To stop a dev server, end the process. On Linux, use `pgrep -x omni` to find it; avoid `pkill -f` patterns, which can match your own shell.
- **Logs:** `omnistore.log` in the data folder. Every command a task ran, and its full output, is on the Activity page.
- **Network:** responses are cached in `cache/http/`. Delete that folder (or use Settings › Clear cache) to force fresh data.

### Things to know

- **Changes to `web/` need a rebuild,** because the files are embedded.
- **The running exe can't be overwritten on Windows.** Quit OmniStore from the tray first, or run the new build from another file name.
- **Elevation:**
  - On Windows, admin commands are written to a `.bat` file and run through `Start-Process -Verb RunAs`; the output is read back from a log file.
  - On Linux, pkexec is used, falling back to sudo with the in-app password prompt.
  - Test both paths when you add admin commands.
- **Kits:** appending data to the exe works, but it breaks an Authenticode signature. If you start signing releases, move the kit into a separate file or a signed resource.
- **No proxy or offline mode beyond the cache:** online catalogs simply return nothing when offline, and the UI shows that per provider.

## 11. Known limits and good next steps

### Limits

- **WinGet installs** need the `winget` command, but search and store pages don't.
- **Windows Update install and the native save dialog** are written against the documented APIs but have only been checked on Windows by hand.
- **The basic translations** cover about 370 texts each. Long explanations stay in English.
- **The Linux tray icon** needs a StatusNotifier host (GNOME needs the AppIndicator extension).
- **GitHub API:** 60 lookups an hour without a token (Settings › GitHub token).

### Ideas for next

- A macOS build. Homebrew is already a provider, and the code is mostly portable; it would need autostart, tray and dialog code for macOS.
- A Microsoft Store provider (msstore source of winget).
- Signed releases, with kits moved to a separate file.
- Central management: read the policy file and curated catalog from a URL, and report installed apps back.
- Completing the basic translations: run `untranslated()` per page and fill in the JSON.
