# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

subconverter is a C++20 utility that converts between proxy subscription formats (Clash, ClashR, Surge 2/3/4, Quantumult, Quantumult X, Loon, Surfboard, Mellow, sing-box, V2Ray, SS/SIP002, SSR, SSD). It runs as an HTTP backend exposing a `/sub` endpoint and can also run as a one-shot artifact generator.

## Build

There is no unified `make` target that installs dependencies; the platform build scripts in `scripts/` install dependencies and then build. Use these for a from-scratch build:

- `scripts/build.macos.release.sh` — macOS (brew deps + builds yaml-cpp, quickjspp, libcron, toml11 from source)
- `scripts/build.alpine.release.sh` — Linux (used by CI via multiarch Alpine docker images)
- `scripts/build.windows.release.sh` — Windows (MSYS2/MinGW)

Once dependencies are installed, the incremental local build is:

```sh
cmake -DCMAKE_BUILD_TYPE=Release .
make -j$(nproc)
```

Dependencies: libcurl, rapidjson, toml11, yaml-cpp (>=0.6.3), PCRE2, QuickJS, libcron. Header-only libs vendored in `include/` (httplib, inja, jpcre2, nlohmann/json).

CMake options: `BUILD_STATIC_LIBRARY` (builds a trimmed static lib with `-DNO_JS_RUNTIME -DNO_WEBGET` — parser/generator only, no server/script), `USING_MALLOC_TRIM`.

## Running

The binary must run from a directory containing the `base/` config files (CI copies `base/` into a `subconverter/` output dir). It listens on `127.0.0.1:25500` by default.

- Server mode (default): `./subconverter` — serves HTTP. Key routes registered in `main.cpp`: `/sub` (main conversion), `/sub2clashr`, `/surge2clash`, `/getruleset`, `/getprofile`, `/render`, `/version`, `/refreshrules`, `/readconf`, `/flushcache`.
- Generator mode: `./subconverter -g` — reads `generate.ini` and writes artifacts (used for CI/offline generation). `--artifact <name>` limits which profiles to generate.
- CLI flags: `-f/--file <pref>` (preferences path), `-l/--log <file>`, `-cfw` (Clash-for-Windows child-process mode).
- Env overrides: `PORT`, `API_MODE`, `MANAGED_PREFIX`, `API_TOKEN`.

There is no test suite in this repository.

## Architecture

The conversion pipeline is: **source subscription → parse into `Proxy` structs → node manipulation → generate target format**.

### The `Proxy` intermediate representation

`src/parser/config/proxy.h` defines the `Proxy` struct and `ProxyType` enum — this is the universal IR that every parser produces and every generator consumes. Any change touching proxy protocols passes through here.

### Parsers (`src/parser/`)

`subparser.cpp` contains one `explode*` function per source format (`explodeVmess`, `explodeSS`, `explodeSSR`, `explodeTrojan`, `explodeClash`, `explodeHysteria2`, etc.), each filling a `Proxy`. Dispatchers tie them together:
- `explode(link, node)` — single node link
- `explodeConfContent(content, nodes)` — a full config/subscription body, detects format
- `explodeSub(sub, nodes)` — subscription (base64 list, Clash YAML, SSD, etc.)

`infoparser.cpp` extracts subscription userinfo (traffic/expiry).

### Generators (`src/generator/`)

`config/subexport.cpp` contains one `proxyTo*` function per target (`proxyToClash`, `proxyToSurge`, `proxyToSingBox`, `proxyToLoon`, `proxyToQuanX`, etc.) that turn a `std::vector<Proxy>` plus a base template + proxy groups + rulesets into the final config string. `config/ruleconvert.cpp` converts rulesets; `config/nodemanip.cpp` preprocesses/filters/renames/sorts nodes. `template/` renders inja/jinja2 templates (`/render` endpoint, template vars from request args).

### Request orchestration (`src/handler/`)

`interfaces.cpp` — `subconverter()` is the `/sub` handler and the heart of the app: it parses ~40 URL args, loads/merges external config, fetches subscriptions, runs the parse → manipulate → generate pipeline, and applies caching/upload. Also holds the other endpoint handlers (`getRuleset`, `getProfile`, `renderTemplate`, `simpleGenerator` for `-g` mode).

`settings.cpp` / `settings.h` — the global `Settings global` struct holds all configuration; `readConf()` loads it from `pref.ini`/`pref.toml`/`pref.yml`. `ExternalConfig` represents a per-request external config file.

`webget.cpp` (HTTP fetch + response cache), `multithread.cpp`, `upload.cpp` (GitHub Gist upload).

### Server & scripting

`src/server/webserver_httplib.cpp` is the active server (the `webserver_libevent.cpp` variant is disabled in CMake). `src/script/` provides a QuickJS runtime (`script_quickjs.cpp`) for user-supplied JS sort/filter scripts and a libcron-based scheduler (`cron.cpp`).

## Configuration files (`base/`)

Runtime config shipped alongside the binary:
- `pref.example.{ini,toml,yml}` — main preferences (three interchangeable formats; `main.cpp` auto-copies an example to `pref.*` on first run).
- `base/base/` — per-target base templates (`clash`/`GeneralClashConfig.yml`, `surge.conf`, `singbox.json`, `loon.conf`, `quanx.conf`, etc.).
- `base/config/` — ACL4SSR rule-set profiles referenced by the `config=` URL arg.
- `base/snippets/` — reusable group/ruleset/emoji/rename definitions (`.txt` and `.toml` variants).
- `base/profiles/` — profile definitions for `/getprofile`.
- `generate.ini` — artifact definitions for generator mode.

## Adding a new proxy protocol

1. Add the type to `ProxyType` enum and `getProxyTypeName()` in `src/parser/config/proxy.h` (plus any new fields on `Proxy` and a `*_DEFAULT_GROUP` macro).
2. Add an `explode*` parser in `subparser.cpp` and wire it into the relevant dispatcher.
3. Add handling in each target's `proxyTo*` generator in `subexport.cpp` (recent commits show this is where per-target support like "Hysteria2 for Loon/sing-box" lands).
