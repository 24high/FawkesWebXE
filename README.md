# FawkesWebXE

FawkesWebXE is a privacy-focused fork of [Tor Browser](https://www.torproject.org/).
Almost everything it adds works inside the Gecko engine: the network stack, the
JavaScript bindings, the style system and the event system. There is no
extension and nothing is injected into pages from the outside.

The browser keeps Tor Browser's protections (resist fingerprinting, letterboxing,
first-party isolation, per-domain circuits, the security levels and New
Identity) and adds a set of switches that can be set **per website** from a
toolbar button or **globally** in `about:preferences#fawkes`.

- Base: Tor Browser 16.0a14 sources
  (`tor-browser-153.5.0esr-16.0-1-build2`, branch
  `tor-browser-153.5.0esr-16.0-1`).
- Branch: `fawkes`. The complete changes are in `fawkes-patches/`, one phase
  per directory, see [Patch series](#patch-series).
- Platform: Linux, Windows and macOS desktop builds. Android is not supported
  by the Fawkes changes.
- Application name: `fawkeswebxe` (display name `FawkesWebXE`).
- No updater and no update channel: the browser never downloads Tor Browser or
  Firefox updates.

---

## Table of contents

- [Defaults in this fork](#defaults-in-this-fork)
- [How the settings work](#how-the-settings-work)
- [User interface](#user-interface)
- [Protections in detail](#protections-in-detail)
  - [Network](#network)
  - [JavaScript APIs](#javascript-apis)
  - [CSS features](#css-features)
  - [Behavioural profiling](#behavioural-profiling)
  - [Fingerprint randomisation](#fingerprint-randomisation)
  - [Cookie banners](#cookie-banners)
  - [Proxy chain](#proxy-chain)
  - [Proxy scraper](#proxy-scraper)
- [Tor network](#tor-network)
- [Cloudflare challenges](#cloudflare-challenges)
- [Preferences](#preferences)
- [Repository layout](#repository-layout)
- [Building](#building)
  - [Full build with tor-browser-build](#full-build-with-tor-browser-build)
  - [Windows build with WebRTC (MSVC/clang-cl cross build)](#windows-build-with-webrtc-msvcclang-cl-cross-build)
  - [Developer build with mach](#developer-build-with-mach)
  - [Applying the patches to a checkout](#applying-the-patches-to-a-checkout)
- [Tests and lint](#tests-and-lint)
- [Documentation](#documentation)
- [Known limits](#known-limits)
- [Patch series](#patch-series)
- [Upstream and license](#upstream-and-license)

---

## Defaults in this fork

Every protection is **on by default**, for every website, except the proxy
chain. The defaults are defined once in
`toolkit/components/fawkes/FawkesSwitches.inc`; the C++ constant
`kFawkesBuiltinDefaults` and the generated `FawkesSwitches.sys.mjs` are built
from it.

| Area | Default |
|---|---|
| Network | Ad blocker, referer blocker, Googlebot user agent, WebRTC block, HTTP/3 block, local network block: on. Proxy chain: **off**. |
| Privacy and fingerprinting | Cookie banner handling and fingerprint randomisation: on. |
| JavaScript APIs | All categories on (WebGL/WebGPU, canvas, audio, device information, sensors, fonts, storage, beacons, clipboard, hardware interfaces, DRM, pausing background pages, custom patterns). |
| CSS features | All categories on (media queries, local fonts, interaction tracking, scroll-driven animations, environment values, custom patterns). |
| Behavioural profiling | All categories on (scroll, pointer, attention, exit intent, typing dynamics, session replay). |

Further fork defaults:

| Setting | Value |
|---|---|
| Tor network | on, and the browser connects automatically at startup (`torbrowser.settings.quickstart.enabled = true`, `extensions.torlauncher.prompt_at_startup = false`) |
| DNS without Tor | DNS over HTTPS **only**, resolver `https://dns.nextdns.io/` (`fawkes.direct.doh.mode = 3`) |
| Application icon | `fawkes.ico` (see `browser/branding/fawkeswebxe/`) |
| Cloudflare challenge host | `challenges.cloudflare.com` is exempt from the JavaScript, CSS and behavioural profiling protections so that the bot check can run; the site that shows the challenge keeps all protections |
| Proxy judge | optional; without one the browser checks the proxies itself (see [Proxy scraper](#proxy-scraper)) |

Changing a default in the settings removes the per-site overrides that become
equal to it, so the UI always shows the effective value.

---

## How the settings work

### Site key

The settings of a page are those of its **top-level site**: the eTLD+1 of the
top-level document (`https://a.b.example.co.uk/` is `example.co.uk`), or the
host for IP addresses and hosts without a public suffix (`localhost`,
`router`, `127.0.0.1`). Iframes, workers, worklets and subresources inherit the
settings of the top-level site through the partition key of their cookie jar
settings. Only `http`, `https` and `blob:` documents of web origins have
Fawkes settings; `about:` pages, `chrome:`, `resource:` (PDF.js), `file:` pages,
extensions and the browser UI never do.

### Storage

Tor Browser always runs in permanent private browsing, so the per-site
settings live in a file of their own, `fawkes-sites.json` in the profile
directory:

```json
{
  "version": 1,
  "defaults": { "js.canvas": true },
  "sites": {
    "example.com": { "adblock": false, "googlebotUA": true }
  }
}
```

- Switches are stored by their stable ID (`adblock`, `js.canvas`, ...).
- `defaults` only lists switches whose default differs from the built-in one,
  and `sites` only lists sites with at least one deviation, so the file reveals
  as little as possible. Set a switch back to the default and the entry
  disappears.
- The file is written atomically through `JSONFile`/`IOUtils`. A corrupt file
  is kept as `fawkes-sites.json.corrupt` and the browser starts with the
  defaults.
- "New Identity" does **not** delete the file: per-site choices are deliberate
  and persistent. It does reset all session state (randomisation keys, chosen
  proxies, cookie banner counters). The settings page can remove single sites
  or all of them.

### When changes apply

Network switches (ad blocker, referer, Googlebot user agent, HTTP/3, local
network, proxy chain) apply to new requests immediately. JavaScript, CSS and
profiling switches apply to documents loaded afterwards, because a document and
its workers keep the flags they were created with. The toolbar menu therefore
reloads the current tab when it closes and a switch changed
(`fawkes.menu.reloadOnClose`, on by default).

### Flags

All switches are bits in a 64-bit flag set. The parent process computes the
flags for every channel from its top-level site, stores them in `LoadInfo` and
sends them to the content process with the document channel. Documents,
workers and worklets carry the flags; `[Func]` checks in WebIDL, the event
system, the style system and the network stack read them. A flag set of 0 means
"block nothing" and is what exempt contexts get. Details:
`browser/components/fawkes/docs/architecture.md`.

---

## User interface

### Toolbar button and menu

A customizable toolbar button (`fawkes-button`) next to the security level
button opens a menu for the current site:

- the site the settings belong to, and the connection mode (Tor / without Tor);
- **Network**: ad blocker (with the number of blocked requests), referer
  blocker, Googlebot user agent, "Enable WebRTC", HTTP/3 block, local network
  block, and a submenu for the proxy chain (current proxy, failover note,
  "Choose a new proxy");
- **Privacy and fingerprinting**: cookie banners, fingerprint randomisation;
- **JavaScript**, **CSS** and **Behavioural profiling** submenus with one
  checkbox per category, plus "all" for the profiling group;
- "Reset this site to the defaults", "Use these settings as the defaults for
  all sites" and "Fawkes settings...".

The button shows a badge when the site has settings that differ from the
defaults, and a "!" badge while the browser runs without Tor. WebRTC and
scroll-driven animations are shown as disabled when the build or the platform
cannot support them.

### Settings page

`about:preferences#fawkes` contains:

- a note where the settings are stored;
- the Tor connection mode, "Reconnect to Tor automatically" and, without Tor,
  the DNS choice (system resolver, DoH with fallback, DoH only) and the DoH
  resolver;
- the global defaults of every switch, grouped like the menu, with "Reset all
  defaults";
- the list of sites with their own settings ("Remove" per site, "Remove all");
- the Googlebot user agent (modern preset, classic preset, custom string);
- the referer mode ("always" or "only cross-site");
- the proxy chain: the proxy list, latency limit, retry time, the fail-closed
  option, and the proxy scraper with its sources, countries, optional judge and
  progress;
- the status of the `adblock.txt` list and a "Reload" button;
- additional blocked domains and the custom JavaScript/CSS patterns;
- whether the in-tree cookie banner rule list is used.

All strings are localized in English and German
(`browser/components/fawkes/locales/{en-US,de}/fawkes.ftl`).

---

## Protections in detail

The full documentation with hooks, tests and limits is in
`browser/components/fawkes/docs/`. This section is an overview.

### Network

Implemented in `toolkit/components/fawkes/FawkesNetwork*.cpp`,
`FawkesAdblock.cpp` and `FawkesLocalNetwork.cpp`, called from
`nsHttpChannel::AsyncOpen` in the parent process.

- **Ad blocker** (`adblock`): blocks the domains of `adblock.txt` next to the
  program file (or next to the binary in the `Browser/` directory) plus the
  domains in `fawkes.blockedDomains`, including subdomains. Supported lines:
  `domain.tld`, hosts format (`0.0.0.0 domain.tld`), simple ABP domain rules
  (`||domain.tld^`); comments (`#`, `!`, `[Adblock ...]`) and unsupported rules
  are skipped. The list is loaded asynchronously at startup; a missing list
  never blocks the browser. Blocked navigations get their own error page,
  blocked subresources fail, frames stay empty, and the menu shows the count.
  No list is shipped; `docs/adblock.txt.example` documents the format.
- **Referer blocker** (`blockReferer`): no `Referer` header, empty
  `document.referrer`, no referrer for navigations, subresources,
  `window.open()` and workers. `fawkes.referer.mode` selects "always" (default)
  or "only cross-site". The `Origin` header is never touched.
- **Googlebot user agent** (`googlebotUA`): the configured Googlebot string in
  the `User-Agent` header and in `navigator.userAgent`, `appVersion` and
  `platform` (window and worker globals) for all requests of the site. The
  settings offer a modern Chrome-based string, the classic string and a custom
  value.
- **HTTP/3 block** (`blockHttp3`): no QUIC, no evaluation of `Alt-Svc: h3` or
  HTTPS RR `alpn=h3`, no WebTransport. With Tor this is the case anyway; the
  switch matters without Tor, where HTTP/3 is enabled globally.
- **Local network and port scans** (`blockLan`): requests from public sites to
  local and private addresses are blocked: `127/8`, `10/8`, `172.16/12`,
  `192.168/16`, `169.254/16`, `100.64/10`, `0.0.0.0/8`, IPv6 `::1`, `::`,
  `fe80::/10`, `fc00::/7`, IPv4-mapped addresses, alternative IPv4 notations
  (decimal, octal, hex), `localhost`, `*.localhost`, `*.local`, `*.lan`,
  `*.home`, `*.internal`, `*.intranet`, `*.corp`, `*.home.arpa`, single-label
  host names, and Tor's own ports. Blocked requests fail immediately with the
  same error, independent of the port, so timing port scans learn nothing.
  Without Tor the resolved addresses are checked as well. A page that is itself
  local may still reach local targets.
- **Proxy chain** (`proxyChain`): see [Proxy chain](#proxy-chain).

### JavaScript APIs

Implemented in `toolkit/components/fawkes/FawkesBindings.cpp`,
`FawkesDom.cpp`, `FawkesScriptPatterns.cpp` and `FawkesPatterns.cpp`; hooks in
the DOM bindings, `EventListenerManager`, `ScriptLoader` and the window
suspension code. "Web content" only: browser code, extensions and their content
scripts are never affected.

| Switch | What it does |
|---|---|
| `js.webgl` | Removes WebGL/WebGL2/WebGPU interfaces and `navigator.gpu`; `getContext()` returns `null`. |
| `js.canvas` | Canvas readout returns Firefox's randomised placeholder (new pixels on every read) without the canvas prompt; text metrics are rounded. |
| `js.audio` | Removes `OfflineAudioContext`; `AnalyserNode` and audio worklet input return silence. |
| `js.device` | Neutralises `hardwareConcurrency`, `maxTouchPoints`, plugins, screen orientation, gamepads, media devices, speech voices, `navigator.connection`; ignores related events. |
| `js.sensors` | Removes device orientation/motion/light/proximity, geolocation and their events. |
| `js.fonts` | Local font sources of script-created `FontFace` objects are skipped. |
| `js.storage` | Storage lives in memory only: `localStorage`, `sessionStorage`, `indexedDB` (null), Cache API, service workers, `BroadcastChannel`, `SharedWorker`. |
| `js.beacons` | `sendBeacon()` and `keepalive` send nothing, `<a ping>` is ignored, resource timing and the Reporting API are removed. |
| `js.clipboard` | Clipboard and selection events of web content are ignored; `navigator.clipboard.read*()` returns nothing. |
| `js.hardware` | Removes MIDI, serial, Web Share, Push, Wake Lock and WebXR. |
| `js.drm` | Removes EME (`requestMediaKeySystemAccess`, `MediaKeys`, ...). |
| `js.backgroundPause` | Pauses timers, workers, animations, audio contexts and media of background tabs and inactive windows. |
| `js.customPatterns` | Drops inline, external, module and worker scripts whose source matches `fawkes.patterns.js`. |

WebRTC is a network switch (`blockWebRTC`): it removes `RTCPeerConnection` and
the other WebRTC interfaces and hides `navigator.mediaDevices`. It is forced on
while the proxy chain is used and on pages that are not web content. WebRTC is
compiled in on Linux and macOS; the Windows MinGW builds cannot compile it
(Mozilla bug 1393901) and pass `--disable-webrtc` in
`mozconfig-windows-i686`/`mozconfig-windows-x86_64`, so there the menu item is
disabled with "(not included in this version)". For WebRTC on Windows, use the
MSVC/clang-cl cross build (`mozconfig-windows-msvc`, see
[Windows build with WebRTC](#windows-build-with-webrtc-msvcclang-cl-cross-build)).

### CSS features

Implemented in `toolkit/components/fawkes/FawkesCss.cpp` and
`servo/components/style/fawkes.rs`, in the style system and the CSS loader.
Only author style sheets of web content are affected.

| Switch | What it does |
|---|---|
| `css.mediaQueries` | Device and preference media features report the resistFingerprinting values (`resolution`, `device-width`, `color`, `prefers-*`, `pointer`, `hover`, `forced-colors`, `display-mode`, ...), consistent with what scripts see. |
| `css.localFonts` | `local()` sources in `@font-face` are ignored, so installed fonts cannot be enumerated. |
| `css.interaction` | Images that only appear in interaction states (`:hover`, `:active`, `:focus*`, `:checked`, `:target`, `[value]`, validity, ...) are never loaded. |
| `css.scrollAnimations` | Scroll- and view-driven animations get no timeline (always on where Gecko does not support them). |
| `css.env` | `env(safe-area-inset-*)` is 0; other `env()` values are not exposed to sites. |
| `css.customPatterns` | Drops style sheets whose source matches `fawkes.patterns.css`. |

### Behavioural profiling

Implemented in `FawkesDom.cpp`, `FawkesTiming.cpp` and
`FawkesSessionReplayDomains.inc`. Listeners, `on*` properties and attributes of
web content are ignored for the affected events; browser-internal handling
keeps working.

| Switch | What it does |
|---|---|
| `profiling.scroll` | Ignores `scroll`, `scrollend`, `wheel` and related events; `IntersectionObserver` reports only the first state; lazy-loaded images and frames load immediately. |
| `profiling.pointer` | Ignores mouse, pointer and touch movement events. |
| `profiling.attention` | Hides `visibilitychange`, `pagehide` and window/document focus from the page; `document.hidden` is `false`, `visibilityState` is `"visible"`, `hasFocus()` is `true`. |
| `profiling.exitIntent` | Hides mouse-leave events at the top-level document and suppresses `beforeunload` dialogs. |
| `profiling.typing` | While the user types, `performance.now()`, `Date`, `requestAnimationFrame` timestamps and keyboard/input event timestamps are coarsened (100 ms plus jitter; `fawkes.typing.*`). |
| `profiling.sessionReplay` | Blocks the requests of known session replay and heatmap providers (Hotjar, Microsoft Clarity, FullStory, Mouseflow, Smartlook, LogRocket, Contentsquare, Yandex Metrica Webvisor, Lucky Orange, Crazy Egg, Inspectlet, ...) in the parent, in addition to the ad blocker. |

Polling scroll positions, element sizes or the hover state with timers cannot
be prevented without breaking most sites. The switches remove the convenient
signals, they do not make profiling impossible.

### Fingerprint randomisation

Implemented in `FawkesRandomize.cpp` and the matching code in the DOM and style
system. A random session key is combined with the site to derive a
deterministic value per (session, site): stable within a site and session, new
after a restart or New Identity. Randomised are canvas/WebGL/audio noise, WebGL
vendor/renderer strings and extensions, screen and window size,
`devicePixelRatio`, colour depth and the matching media features,
`hardwareConcurrency`, `maxTouchPoints`, cameras, fonts asked for by name,
element and text boxes, the time zone, and the user agent within the operating
system family. Such sites always get English (`Accept-Language`,
`navigator.languages`, `Intl`). Blocking switches and the Googlebot user agent
take precedence.

### Cookie banners

`cookiebanners.service.mode = 2` ("reject, otherwise accept") with the rules
from the in-tree Remote Settings dump (no network requests) plus Fawkes rules,
for normal and private browsing, also in frames. A heuristic
JSWindowActor (`FawkesCookieBanner`) handles unknown banners in every frame and
in open and closed shadow roots: it clicks "reject all"/"only necessary" when
possible, otherwise turns off every optional purpose in the settings and saves,
and accepts only when there is no way to reject ("pay or OK"). Scroll locks are
released afterwards. The switch `cookieBanner` decides per site; the rule list
can be turned off (`fawkes.cookiebanners.useRuleList`), then only the
heuristics run.

### Proxy chain

`proxyChain` sends every request of a site through Tor and then through an
external SOCKS5 proxy from `fawkes.proxy.list`; without Tor it is a single hop.
Each site gets a random proxy for the session (visible in the menu, changeable
with "Choose a new proxy"; New Identity resets it). The second SOCKS5 handshake
happens in the SOCKS layer (`netwerk/socket/nsSOCKSIOLayer.cpp`), so target
names stay names and nothing is resolved locally. A proxy that is unreachable
or slower than `fawkes.proxy.maxLatencyMs` (default 1000 ms) is skipped for
`fawkes.proxy.retryAfterSeconds` (default 300 s). If no proxy works the request
fails closed with its own error page unless "continue without an external
proxy" (`fawkes.proxy.fallbackDirect`) is enabled. `.onion` addresses and local
hosts bypass the chain, and HTTP/3, WebTransport and WebRTC are off while it is
active.

Proxy list format, one proxy per line:

```
# comments start with #
203.0.113.10:1080
user:password@203.0.113.11:1080
socks5://user:password@[2001:db8::1]:1080
socks5h://203.0.113.12:1080
```

### Proxy scraper

The scraper sits next to the proxy list and only runs on a click. It fetches
eight public SOCKS5 list sites (FreeProxy.World, ProxyScrape, ProxyHub,
Geonode, ProxyDB, ProxyListPlus, spys.me, My-Proxy.com; see
`docs/proxy-sources.md`), validates every entry (IP literals only, no private,
local or reserved addresses, no Tor ports), looks up the country in tor's
offline `geoip`/`geoip6` databases, and checks the proxies the same way the
chain would use them:

1. the listed address is in a known, unblocked country (default block list:
   the "14 Eyes");
2. three SOCKS5 greetings through the chain, median below
   `fawkes.proxy.maxLatencyMs`, and a successful `CONNECT`;
3. an HTTPS request through the proxy passes normal TLS validation.

With a **proxy judge** configured (`fawkes.proxy.scraper.judgeUrl`, an
`https://` page that shows the client address and request headers, e.g.
httpbin's `/get`), the scraper additionally checks that the judge sees neither
proxy headers nor the user's or the Tor exit's address, and that the outgoing
address is in a known, unblocked country.

Without a judge (the default) the browser checks the proxies itself: a
request over valid TLS to `https://example.com/` through the proxy has to work,
and the address checks are skipped. This is the "internal recognition" mode
that needs no external service.

The scraper appends the proxies that pass to the list below a comment line with
the date, without duplicates, fastest first. "Check the existing list again"
rechecks the list and marks failed entries with a `# [Fawkes] ...` comment
(nothing is removed; the notes disappear when a proxy passes again). Cancelling
closes all connections at once. The GeoIP data comes from tor's files; the
scraper keeps nothing but its progress in memory.

---

## Tor network

Tor is on by default and the browser connects automatically at startup: the
"Always connect automatically" setting on `about:torconnect`
(`torbrowser.settings.quickstart.enabled = true`) fires on the first start too,
because `extensions.torlauncher.prompt_at_startup` is `false`. The mode is
global, never per site: mixing sites with and without Tor would link the real
address to the Tor identity.

- **Switching** (settings or "Continue without Tor..." on `about:torconnect`)
  asks for confirmation, deletes the session like New Identity (tabs, cookies,
  caches, site data, permissions) and restarts the browser. Only then is
  `fawkes.tor.enabled` stored. Environment variables that configure tor
  (`TOR_PROVIDER`, `TOR_SKIP_LAUNCH`, `TOR_SOCKS_HOST`, ...) always keep Tor
  on; no variable turns it off.
- **Without Tor** no tor process and no control port are started and
  `about:torconnect` is skipped. The direct connection is configured and
  locked at startup (the next Tor session has nothing to undo).
- **`.onion`** addresses are refused before DNS and before any connection, with
  their own error page and without a retry button. Onion-Location and the onion
  search are hidden.
- **DNS without Tor**: system resolver, DoH with fallback, or **DoH only**
  (the default, resolver `https://dns.nextdns.io/`). DoH-only without a valid
  resolver fails closed. The DoH provider sees every domain and the user's
  address. With Tor the exit resolves names and local DNS stays disabled.
- **Visibility**: a permanent "Without Tor - your IP address is visible to
  websites" warning in the address bar and in the menu, a "!" badge on the
  Fawkes button, and the Tor-only UI (circuit display, new circuit, Connect
  button, bridges) is hidden.
- **Reconnect** (`fawkes.tor.autoReconnect`, off by default): after a
  successful connection in the session, restarts tor and reconnects after 5 s,
  30 s and 2 min if the connection drops. It never changes the mode, the proxy
  or the bridges, and never starts Connection Assist.

---

## Cloudflare challenges

Sites behind Cloudflare often show a bot check served from
`https://challenges.cloudflare.com`. With all protections on, such a check
would be blocked like any other script. Fawkes therefore treats documents of
that host as exempt: the JavaScript, CSS and behavioural profiling bits are
cleared for them, so the challenge can run. The site that shows the challenge
keeps its own protections, so its own scripts stay blocked.

The exemption is implemented in `FawkesFlags.cpp`
(`ComputeDocumentFlags()` and `FawkesGroupSwitchMask()`); it is a built-in
default, not a user setting.

---

## Preferences

Most settings are per site and live in `fawkes-sites.json`. The global ones
are preferences (defaults in `browser/components/fawkes/fawkes-prefs.js`, and
`fawkes.googlebot.userAgent`, `fawkes.referer.mode`, `fawkes.typing.*` as
static prefs in `modules/libpref/init/StaticPrefList.yaml`).

| Preference | Default | Meaning |
|---|---|---|
| `fawkes.tor.enabled` | `true` | Use the Tor network; read once at startup. |
| `fawkes.tor.autoReconnect` | `false` | Restart tor and reconnect when the connection drops. |
| `fawkes.direct.doh.mode` | `3` | DNS without Tor: 0 system resolver, 2 DoH with fallback, 3 DoH only. |
| `fawkes.direct.doh.uri` | `https://dns.nextdns.io/` | The DoH resolver (`https://` only). |
| `fawkes.googlebot.userAgent` | modern Googlebot string | User agent used by the `googlebotUA` switch. |
| `fawkes.referer.mode` | `0` | 0 always remove the referrer, 1 only cross-site. |
| `fawkes.menu.reloadOnClose` | `true` | Reload the tab when the menu closes and a switch changed. |
| `fawkes.blockedDomains` | empty | Extra blocked domains (newline separated). |
| `fawkes.patterns.js` / `.css` | empty | Custom script/style sheet patterns (substring or regular expression, newline separated). |
| `fawkes.cookiebanners.useRuleList` | `true` | Use the in-tree cookie banner rules in addition to the heuristics. |
| `fawkes.proxy.list` | empty | External SOCKS5 proxies for the proxy chain. |
| `fawkes.proxy.maxLatencyMs` | `1000` | Greeting time limit of the proxy chain (0: no limit). |
| `fawkes.proxy.retryAfterSeconds` | `300` | How long a failed or slow proxy is skipped. |
| `fawkes.proxy.fallbackDirect` | `false` | Continue without an external proxy instead of failing closed. |
| `fawkes.proxy.scraper.judgeUrl` | empty | Optional HTTPS proxy judge; without one the browser checks the proxies itself. |
| `fawkes.proxy.scraper.blockedCountries` | `US,GB,CA,AU,NZ,DK,FR,NL,NO,DE,BE,IT,ES,SE` | Countries whose proxies are never added. |
| `fawkes.proxy.scraper.sources` | all eight sites | The list sites that are searched. |
| `fawkes.proxy.scraper.parallelChecks` | `32` | Proxies checked at the same time (1-64). |
| `fawkes.proxy.scraper.maxCandidates` | `1000` | The most proxies checked in one search. |
| `torbrowser.settings.quickstart.enabled` | `true` | "Always connect automatically". |
| `extensions.torlauncher.prompt_at_startup` | `false` | Allows the quickstart to fire on the first start too. |

---

## Repository layout

| Path | Content |
|---|---|
| `toolkit/components/fawkes/` | Engine code: switch table, `Services.fawkes`, network, JavaScript, CSS, profiling, randomisation, cookie banners, proxy chain and scraper, Tor mode. |
| `browser/components/fawkes/` | UI: toolbar button and menu (`Fawkes.sys.mjs`), `about:preferences#fawkes`, cookie banner actors, localisation, tests, documentation. |
| `browser/branding/fawkeswebxe/` | Branding: name, icons, logos, about graphics. |
| `tools/fawkes/` | `make-placeholder-icons.py` and the branding check script. |
| `fawkes-patches/` | The complete change series, one directory per phase plus one combined patch per phase (see [Patch series](#patch-series)). |
| `browser/components/fawkes/docs/` | Technical documentation for every feature. |

---

## Building

There are two ways to build FawkesWebXE. Use **tor-browser-build** for a
runnable, packaged browser (also for Windows and macOS); use a **mach
developer build** for quick iteration on Linux.

### Full build with tor-browser-build

This is how the official Tor Browser is built. The result contains the tor
binary, the pluggable transports, the packaging and the installer.

**Requirements (Linux, WSL2 works):**

- a recent Linux distribution with `user_namespaces(7)` enabled (`uidmap`
  package, `newuidmap`/`newgidmap`);
- `git`, `zstd`, `zip`, `jq`, `make`, and Perl with the modules listed in the
  `tor-browser-build` `README`. On Debian and Ubuntu:

  ```bash
  sudo apt install libdata-dump-perl libdata-uuid-perl libdatetime-perl \
    libdigest-sha-perl libfile-copy-recursive-perl libfile-slurp-perl \
    libio-all-perl libcapture-tiny-perl libio-handle-util-perl libjson-perl \
    libparallel-forkmanager-perl libpath-tiny-perl libsort-versions-perl \
    libstring-shellquote-perl libtemplate-perl libxml-libxml-perl \
    libxml-writer-perl libyaml-libyaml-perl git uidmap zstd jq zip
  ```
- a container runtime usable by rbm (Podman, Docker, or rbm's own
  user-namespace based setup);
- on Ubuntu 24.04 and later:
  `sysctl -w kernel.apparmor_restrict_unprivileged_userns=0`;
- enough disk space (several GB per platform) and, for Windows, a 64-bit
  Windows cross-toolchain that tor-browser-build provides itself.

**Steps:**

```bash
# 1. Get tor-browser-build (the branch that matches the base).
git clone https://gitlab.torproject.org/tpo/applications/tor-browser-build.git
cd tor-browser-build
git checkout tbb-16.0a14-build2        # or the branch you build from
make submodule-update                  # fetches rbm/

# 2. Apply the FawkesWebXE packaging: product name, program file, installer
#    and disk image graphics, package and start script names.
git am /path/to/tor-browser/fawkes-patches/phase-01-branding/tor-browser-build/*.patch
# or: git apply /path/to/tor-browser/fawkes-patches/phase-01-branding-tor-browser-build.patch

# 3. Point the build at the source tree and the commit to build.
#    projects/firefox/config:
#      git_url: /path/to/tor-browser
#      git_hash: <full commit hash, e.g. git -C /path/to/tor-browser rev-parse fawkes-phase-12>
#    The fork is pinned to a commit, so commit your changes first.

# 4. Build a test build (en-US only, no MAR, fastest).
RBM_NO_DEBUG=1 make torbrowser-testbuild-windows-x86_64
RBM_NO_DEBUG=1 make torbrowser-testbuild-linux-x86_64
RBM_NO_DEBUG=1 make torbrowser-testbuild-macos

# Or a full alpha build:
make torbrowser-alpha-windows-x86_64
```

The result is written below `testbuild/` (or `torbrowser/alpha/unsigned/` for
full builds). `RBM_NUM_PROCS=8` limits the parallelism, `RBM_NO_DEBUG=1`
prevents an interactive shell from opening when a build fails.

**Pinning the source.** `projects/firefox/config` contains the whole source
selection for the browser:

```yaml
git_url: /path/to/tor-browser
git_hash: <full commit hash of fawkes-phase-12>
```

`git_hash` must name a commit that exists in `git_url`; the FawkesWebXE branch
and its phase tags (`fawkes-base`, `fawkes-phase-01` ... `fawkes-phase-12`) can
be used for that:

```bash
git -C /path/to/tor-browser rev-parse fawkes-phase-12
```

Update the hash after every new commit, otherwise the build silently uses the
old source.

**Icons and packaging.** The installer icons and the macOS disk image come from
`tor-browser-build` (`projects/browser/windows-installer/*.ico`,
`projects/common/dmg-root/`). Running
`tools/fawkes/make-placeholder-icons.py --tor-browser-build <dir>` regenerates
the FawkesWebXE placeholders there. The application icon of the built browser
itself is `browser/branding/fawkeswebxe/*.ico` (all variants currently contain
`fawkes.ico`).

### Windows build with WebRTC (MSVC/clang-cl cross build)

The MinGW Windows builds of tor-browser-build cannot include WebRTC (Mozilla
bug 1393901). If you need WebRTC on Windows, cross-compile the Windows target
with MSVC from Linux, the way Mozilla builds Windows on Linux.
`mozconfig-windows-msvc` uses `--target=x86_64-pc-windows-msvc` and
`--enable-bootstrap`, which fetches the "vs" toolchain (MSVC and the Windows
SDK) and clang from Mozilla's taskcluster.

```bash
rustup target add x86_64-pc-windows-msvc
MOZCONFIG=$PWD/mozconfig-windows-msvc ./mach build
```

The result is `obj-x86_64-pc-windows-msvc/dist/bin/fawkeswebxe.exe`.

Requirements and quirks:

- `msitools` (with `libmsi`) on the host: the bootstrapper unpacks the Windows
  SDK MSIs with `msiextract`.
- a Wine that runs Windows PE files: the SDK's `fxc.exe` (shader compiler) runs
  under it. A 64-bit-only Wine needs a patched 32-bit loader or the 32-bit
  glibc (`/lib/ld-linux.so.2`).
- `export MIDL=$HOME/.mozbuild/wine/bin/widl`: the SDK's `midl.exe` is a
  32-bit program that may not run under a 64-bit-only Wine.
- `--enable-bootstrap` needs network access; it downloads the VS sysroot,
  clang, node, nsis, upx and the other toolchains.
- The build is larger and slower than the MinGW build; the first run takes
  tens of minutes.

### Developer build with mach

For quick iteration on Linux. The repository ships a development mozconfig that
already uses the FawkesWebXE branding and enables tests:

```bash
cd /path/to/tor-browser

# Dependencies (Rust, clang, Node, ...).
./mach bootstrap

# Linux x86_64 development build.
cp mozconfig-linux-x86_64-dev mozconfig
./mach build
./mach run
```

To use a mozconfig of your own, the relevant options are:

```bash
ac_add_options --enable-application=browser
ac_add_options --with-branding=browser/branding/fawkeswebxe
ac_add_options --with-app-name=fawkeswebxe
ac_add_options --disable-updater
ac_add_options --enable-tests          # for running the test suites
```

A plain mach build does not start tor and is not packaged like a Tor Browser
release; it is meant for developing the Fawkes features. Use tor-browser-build
for anything you want to run as a browser.

### Applying the patches to a checkout

The series is based on `fawkes-base` (= `fawkes-phase-00`). Apply the phases in
order.

With commits (series, `git am`):

```bash
git checkout -b fawkes fawkes-base
git am fawkes-patches/phase-01-branding/*.patch
git am fawkes-patches/phase-02-framework/*.patch
git am fawkes-patches/phase-03-no-tor/*.patch
git am fawkes-patches/phase-04-network/*.patch
git am fawkes-patches/phase-05-js-apis/*.patch
git am fawkes-patches/phase-06-css/*.patch
git am fawkes-patches/phase-07-profiling/*.patch
git am fawkes-patches/phase-08-randomization/*.patch
git am fawkes-patches/phase-09-cookie-banners/*.patch
git am fawkes-patches/phase-10-proxy-chain/*.patch
git am fawkes-patches/phase-11-proxy-scraper/*.patch
git am fawkes-patches/phase-12-customization/*.patch
```

Without commits (combined files, `git apply`):

```bash
git apply fawkes-patches/phase-01-branding.patch
git apply fawkes-patches/phase-02-framework.patch
git apply fawkes-patches/phase-03-no-tor.patch
git apply fawkes-patches/phase-04-network.patch
git apply fawkes-patches/phase-05-js-apis.patch
git apply fawkes-patches/phase-06-css.patch
git apply fawkes-patches/phase-07-profiling.patch
git apply fawkes-patches/phase-08-randomization.patch
git apply fawkes-patches/phase-09-cookie-banners.patch
git apply fawkes-patches/phase-10-proxy-chain.patch
git apply fawkes-patches/phase-11-proxy-scraper.patch
git apply fawkes-patches/phase-12-customization.patch
```

`patch -p1` cannot apply the binary graphics of phase 01, the binary test
certificate of phase 11 or the binary icons of phase 12; use `git apply` for
the combined files.

After applying the series with `git am`, a checkout must be identical to the
phase tag, for example for phase 12:

```bash
git diff --stat fawkes-phase-12   # no output: the tree matches
```

The same check was run for the combined files against `fawkes-base`; the
result is `fawkes-phase-12` (see `fawkes-patches/README.md`).

---

## Tests and lint

The Fawkes tests run like normal Gecko tests. A build with tests enabled is
required (`mozconfig-linux-x86_64-dev` sets `--enable-tests`).

```bash
# xpcshell tests (switch table, storage, services, network rules, proxy parser,
# proxy chain, proxy scraper, cookie banner rules, GeoIP, ...).
./mach test toolkit/components/fawkes/test/xpcshell

# Browser chrome tests (toolbar button, menu, settings page, per-site
# persistence, every JS/CSS/profiling category, randomisation, cookie banners,
# proxy chain, proxy scraper).
./mach test browser/components/fawkes/test/browser

# Lint all changed files.
./mach lint
```

The proxy scraper tests use mock SOCKS5 proxies and mock TLS judges
(`toolkit/components/fawkes/test/FawkesMockSocks.sys.mjs`,
`FawkesMockJudge.sys.mjs`); no test contacts real list sites or judges. The
branding check script (`tools/fawkes/check-branding.py`) needs a finished build
and checks that no user-visible "Tor Browser"/"Firefox" string is left.

---

## Documentation

Technical documentation for every feature lives next to the code:

| Document | Content |
|---|---|
| `browser/components/fawkes/docs/architecture.md` | Components, site key, flag transport, startup. |
| `.../network.md` | Referer, Googlebot UA, HTTP/3, ad blocker, local network. |
| `.../js-apis.md` | JavaScript categories, hooks, event listeners, background pause, WebRTC. |
| `.../css.md` | Media queries, fonts, interaction images, scroll animations, `env()`. |
| `.../profiling.md` | Scroll, pointer, attention, exit intent, typing, session replay. |
| `.../randomization.md` | Session keys, randomised values, English forcing. |
| `.../cookie-banners.md` | Rule list, order of decisions, heuristics. |
| `.../tor-mode.md` | Running without Tor, switching, `.onion`, DNS, warnings. |
| `.../proxy-chain.md` | Proxy list, per-site choice, SOCKS handshakes, failover. |
| `.../proxy-scraper.md` | Proxy checks, judge, sources, limits. |
| `.../proxy-sources.md` | The eight list sites and the rejected candidates. |
| `.../storage.md` | `fawkes-sites.json`, privacy, preferences. |
| `.../categories.md` | Every switch with mechanism and breakage risk. |
| `fawkes-patches/README.md` | Patch layout, apply order, verification. |

The original specification of the fork is `FAWKES_PROMPT.md` (in German).

---

## Known limits

- **Ad blocker**: no list is shipped. Without `adblock.txt` nothing is blocked
  by the list; the settings and the menu show the status.
- **Public proxies** are run by strangers, are short-lived and can log
  connection data. Use the proxy chain only for HTTPS sites.
- **DoH** shows every domain to the resolver (NextDNS by default); with Tor the
  exit resolves names and local DNS stays off.
- **Proxy chain**: names are resolved by the external proxy, and the Tor hop
  keeps the per-site circuit. A proxy that fails over is replaced for the
  session.
- **Profiling**: polling scroll positions or element sizes with timers cannot be
  stopped without breaking most sites.
- **Cloudflare**: the exemption covers documents served from
  `challenges.cloudflare.com`; scripts that a normal page includes directly
  from that host are still subject to the site's switches.
- **Proxy scraper without a judge** cannot check the outgoing address, so the
  address and elite checks are skipped; configure a judge to enable them.
- **Direct mode**: without Tor the transport is not anonymised. The browser
  says so permanently.
- **WebRTC on Windows**: the MinGW builds (all Windows builds from
  tor-browser-build) cannot compile WebRTC, so the "Enable WebRTC" menu item is
  disabled there. Linux and macOS builds include it; for Windows use the
  MSVC/clang-cl cross build (`mozconfig-windows-msvc`) or a native MSVC build.

---

## Patch series

| Phase | Content |
|---|---|
| 00 | `fawkes-base`: the unmodified Tor Browser sources. |
| 01 | Branding: FawkesWebXE name, icons, locales, updater disabled, packaging. |
| 02 | Framework: switch table, `Services.fawkes`, `fawkes-sites.json`, flags parent to content, toolbar button, settings page. |
| 03 | Running without Tor: direct provider, `.onion` block, DoH, warnings, reconnect. |
| 04 | Network: referer, Googlebot UA, HTTP/3, ad blocker, local network. |
| 05 | JavaScript API categories and WebRTC. |
| 06 | CSS categories. |
| 07 | Behavioural profiling. |
| 08 | Fingerprint randomisation and English forcing. |
| 09 | Cookie banners. |
| 10 | Proxy chain. |
| 11 | Proxy scraper. |
| 12 | Fork defaults: icon, DoH, all switches on, Cloudflare exemption, automatic Tor connection, judge-free proxy checks, this README. |

Each phase has a tag `fawkes-phase-NN`, a directory
`fawkes-patches/phase-NN-<name>/` with one patch per commit, and a combined
file `fawkes-patches/phase-NN-<name>.patch`. The tags and the patch directory
are local; nothing is pushed.

---

## Upstream and license

FawkesWebXE is based on Tor Browser and Mozilla Firefox. The upstream
repositories and their documentation:

- [Tor Browser repository overview](https://gitlab.torproject.org/tpo/applications/wiki/-/wikis/Development-Information/Tor-Browser/Tor-Browser-Repository-Overview)
- [Contributing to Tor Browser](https://gitlab.torproject.org/tpo/applications/wiki/-/wikis/Development-Information/Tor-Browser/Contributing-to-Tor-Browser)
- [Mozilla Firefox](https://www.mozilla.org/firefox/)

The code is licensed under the Mozilla Public License 2.0 unless a file says
otherwise. Tor, Tor Browser and Firefox are projects of their respective
organisations; the names are used here only to describe the origin of the
code.
