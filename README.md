# FawkesWebXE

FawkesWebXE gets a **randomized fingerprint** in the EFF [Cover Your Tracks](https://coveryourtracks.eff.org/) test.

![Result of the EFF Cover Your Tracks test: the browser has a randomized fingerprint](browser/components/fawkes/docs/images/eff-cover-your-tracks.png)

FawkesWebXE is a fork of Tor Browser 16.0, based on Firefox 153.5 ESR. It keeps the Tor network, the Tor circuit isolation and the protections of Tor Browser, and adds protection against browser fingerprinting and behavioral profiling, a proxy chain, an ad blocker and automatic cookie banner rejection. You can turn every measure on or off per site.

The protections run in the engine: in the DOM bindings, the event system, the Stylo style system and the network stack. Nothing is injected into pages, and no extension is involved.

- [What is different from Tor Browser](#what-is-different-from-tor-browser)
- [Per-site switches](#per-site-switches)
- [Anti-fingerprinting](#anti-fingerprinting)
- [Anti-profiling](#anti-profiling)
- [Proxy chain](#proxy-chain)
- [Cookies are opt-in](#cookies-are-opt-in)
- [Cookie banners are rejected automatically](#cookie-banners-are-rejected-automatically)
- [Built-in ad blocker](#built-in-ad-blocker)
- [Other network protection](#other-network-protection)
- [Running without Tor](#running-without-tor)
- [Building and testing](#building-and-testing)
- [Documentation](#documentation)

## What is different from Tor Browser

| | Tor Browser | FawkesWebXE |
|---|---|---|
| Fingerprint | The same values for every user (resistFingerprinting) | Values that are random per site and session, but stable on a site |
| Behavioral profiling | No specific protection | Scroll, pointer, attention, exit intent and typing rhythm can be hidden; session replay services are blocked |
| Route | Tor | Tor, or Tor and then an external SOCKS5 proxy per site |
| Cookies | Allowed for every site until the session ends | Blocked until you opt in per site |
| Cookie consent | Left to the user | Rejected for you |
| Cookie banners | Shown | Rejected automatically, with rules and heuristics |
| Ad blocking | None | Domain blocklist from `adblock.txt` and your own list |
| Settings | Global security level | Global defaults and per-site switches in the toolbar |

## Per-site switches

The FawkesWebXE toolbar button opens a menu for the site of the current tab. The site is the registrable domain of the top-level page (eTLD+1), so `www.example.com` and `shop.example.com` share their switches.

- The menu groups the switches into network, privacy, JavaScript, CSS and behavioral profiling.
- Closing the menu reloads the tab, so that the new switches apply to the page.
- `about:preferences#fawkes` sets the defaults for all sites, the proxy list, the ad blocker and the Tor mode.

Tor Browser runs in permanent private browsing mode and forgets site permissions on exit. FawkesWebXE keeps the per-site switches in its own file, `fawkes-sites.json` in the profile directory. Browser pages, extensions and PDF.js are never affected by the switches.

## Anti-fingerprinting

Tor Browser gives every user the same screen size, fonts, user agent and other values, so that users look alike. FawkesWebXE can go a different way: with **Randomize fingerprint** (on by default), each site gets its own set of values.

- Values are stable on a site. Reading a value again, reloading or opening another page of the same site gives the same result, so a site sees no noise it could filter out.
- Values differ between sites, so two sites cannot match their visitors by them.
- Values are new after a restart or **New Identity**.

The randomization covers:

- canvas, WebGL and audio output, and the WebGL vendor, renderer and extensions
- screen and window size, `devicePixelRatio`, color depth and the matching media features
- `hardwareConcurrency`, `maxTouchPoints` and the list of cameras
- fonts that a page asks for by name, and the size of element and text boxes
- the user agent, within the family of your operating system
- the time zone

The language is always English (`Accept-Language`, `navigator.languages` and `Intl`), so the language does not narrow you down.

You can also block fingerprinting surfaces per site instead of randomizing them:

| Switch | Default | What it does |
|---|---|---|
| WebGL and WebGPU | on | Removes the interfaces; `getContext("webgl")` returns `null` |
| Canvas readout | off | Readouts return random pixels, text metrics are rounded |
| Audio fingerprinting | off | Removes `OfflineAudioContext`, silences analyser data |
| Device information | off | Fixed core count, no touch points, plugins, gamepads, devices or voices |
| Sensors and geolocation | on | Removes the sensor events and `navigator.geolocation` |
| Local fonts | on | Skips `local()` sources in `@font-face` and `FontFace` |
| Media queries | off | Fixed values for device size, color scheme, pointer, contrast and others |
| Storage supercookies | off | Keeps `localStorage` and `sessionStorage` in memory, removes `indexedDB`, caches and service workers |
| Beacons | on | `sendBeacon()`, `<a ping>` and resource timing entries send or show nothing |
| Hardware interfaces | on | Removes MIDI, Serial, Web Share, Push, Wake Lock and WebXR |
| WebRTC | on | Removes WebRTC; the menu shows it as "Enable WebRTC" |

Blocked interfaces are removed, not broken, so feature detection with `typeof` keeps working.

Note: A randomized profile is consistent, but it is not the profile that all other Tor Browser users share. Sites see an unusual set of values that changes with every session.

## Anti-profiling

Behavioral profiling builds a picture of you from how you use a page: how fast you read, where the mouse hesitates, when you switch tabs, how you type. The group **Block behavioral profiling** hides these signals per site.

| Switch | Default | Signal it hides |
|---|---|---|
| Scrolling | off | How far and how fast you read: scroll and wheel events, `IntersectionObserver`, lazy loading |
| Pointer movement | off | Mouse and touch movement and hesitation |
| Attention | off | Whether the page is visible and focused, and for how long |
| Exit intent | on | The mouse leaving the page; "Do you really want to leave?" dialogs |
| Typing rhythm | on | Keystroke timing, used for typing biometrics |
| Session replay | on | Requests to session replay and heatmap services |

- **Typing rhythm**: while you type, and for 2 seconds after each key, the page clocks (`performance.now()`, `Date.now()`, event timestamps) only tick in steps of about 100 ms, with jitter. Keyboard and input events always have coarse timestamps.
- **Attention**: the page always looks visible and focused to its scripts. The background pause and the browser itself still see the real state.
- **Session replay**: the built-in list covers Hotjar, Microsoft Clarity, FullStory, Mouseflow, Smartlook, LogRocket, Contentsquare, Yandex Metrica Webvisor, Lucky Orange, Crazy Egg, Inspectlet and others.
- **Pause background pages** (off by default) stops timers, workers, animations and media of pages in background tabs and inactive windows.

Note: A page can still poll scroll positions or element sizes with timers. The switches take away the convenient signals, but they cannot make profiling impossible without breaking most sites.

## Proxy chain

With **Proxy chain (Tor → external proxy)** (off by default), every request of a site goes through Tor and then through an external SOCKS5 proxy from your list:

```
browser → Tor → external SOCKS5 proxy → website
```

The website sees the address of the external proxy instead of a Tor exit, and the external proxy sees a Tor exit instead of you. Tor's circuit isolation per site stays in place.

- The proxy is chosen at random per site and kept for the session.
- A proxy that fails or answers too slowly is replaced by the next one of the list.
- The chain fails closed: if no proxy works, requests fail instead of leaving without the chain.
- HTTP/3, WebTransport and WebRTC stay off on the site, so no traffic can go around the chain.
- `.onion` addresses and local hosts do not use the external proxy.
- Without Tor, the chain has one hop: browser → external proxy → website.

You can manage the proxy list in `about:preferences#fawkes`. The **proxy scraper** next to the list searches public proxy lists for SOCKS5 proxies, checks each one the way the chain would use it, and adds the ones that pass.

- It only runs when you click **Search proxies…** or **Check the existing list again**, and leaving the page cancels it.
- It needs a proxy judge, an `https://` URL that you set yourself. No judge is preset.
- Proxies in the "14 Eyes" countries are left out by default. You can change the list of blocked countries.

Note: Public proxies are run by strangers, often disappear after a short time, and can log connection data or be honeypots. Use the proxy chain only for HTTPS sites.

## Cookies are opt-in

Sites cannot set cookies until you allow it. You opt in per site, and every other site gets no cookies.

FawkesWebXE also never gives consent on your behalf. When a site asks for cookie consent, the answer is "no" unless you decide otherwise.

- A banner is rejected. If it only offers settings, every optional purpose is turned off and saved.
- If a site offers no way to reject ("pay or OK"), the banner is accepted only so that the page can be used.
- To give consent on a site, turn off **Reject cookie banners automatically** for that site and answer the banner yourself.
- With **Storage supercookies**, `localStorage` and `sessionStorage` only live in memory for the page, and `indexedDB`, caches and service workers are removed. This also stops sites from storing an identifier outside of cookies.

## Cookie banners are rejected automatically

**Reject cookie banners automatically** is on by default. Two parts work together:

- Firefox' cookie banner service handles the known consent management platforms and sites, with the rules that ship with the source. The rules are read offline; FawkesWebXE does not fetch them from Mozilla servers.
- Heuristics handle all other banners. They look in every frame, including cross-site frames, and in open and closed shadow roots, and they recognize banner texts in many languages.

After a banner is gone, scroll locks of the page are released. The toolbar menu shows what happened on the current site.

The rules and heuristics cover OneTrust, Cookiebot, Usercentrics, Didomi, Quantcast Choice, Sourcepoint, TrustArc, consentmanager.net, iubenda, Borlabs Cookie, Complianz, CookieYes, Klaro, Google Funding Choices, CookieFirst, Axeptio, Commanders Act, Piwik PRO, Osano, Termly, Ketch and others.

Note: The heuristics can miss a banner or misjudge a button, and sites can tell that their banner was answered automatically.

## Built-in ad blocker

The ad blocker (on by default) blocks the domains of `adblock.txt` and of your own blocked domains, including all their subdomains.

- Put `adblock.txt` in the folder of the program file: `Browser/` on Linux, next to `fawkeswebxe.exe` on Windows, `FawkesWebXE.app/Contents/MacOS/` on macOS.
- The list has one domain per line. See [`adblock.txt.example`](browser/components/fawkes/docs/adblock.txt.example) for the format.
- The list is loaded in the background when the browser starts. **Reload adblock.txt** in the settings loads it again.
- A blocked page shows its own error page; a blocked frame stays empty.
- No list is included, and files over 256 MB are refused.

Blocked requests fail before proxy resolution, DNS or any connection.

## Other network protection

| Switch | Default | What it does |
|---|---|---|
| Block the referer | on | No `Referer` header and an empty `document.referrer`, always or only cross-site |
| Block HTTP/3 | on | No QUIC, no `Alt-Svc: h3`, no WebTransport |
| Block the local network | on | Public sites cannot reach your computer or devices in your network; this stops port scans and router attacks from web pages. Every blocked request fails the same way and at once, so a page cannot tell which ports are open |
| Googlebot user agent | off | Sends the Googlebot user agent in the header and in `navigator` |
| Custom script and stylesheet patterns | off | Drops scripts and stylesheets whose source matches your patterns |

## Running without Tor

FawkesWebXE uses the Tor network by default. You can turn Tor off for the whole browser in the settings, never per site: mixing sites with and without Tor would link your real IP address to your Tor identity.

- Switching takes effect after a restart. Before that, a dialog explains that all tabs, cookies and session data are deleted, like with **New Identity**.
- Without Tor, a warning is shown, the Tor-only parts of the interface are hidden and `.onion` addresses are blocked.
- DNS without Tor uses the system resolver by default. You can set a DNS over HTTPS resolver instead; none is preset.
- **Reconnect to Tor automatically** (off by default) reconnects when the connection to the Tor network is lost.

## Building and testing

FawkesWebXE builds like Tor Browser. The mozconfigs use the FawkesWebXE branding and turn off the updater.

```bash
MOZCONFIG=mozconfig-linux-x86_64-dev ./mach build
```

```bash
./mach run
```

To run the tests of the Fawkes components:

```bash
./mach test toolkit/components/fawkes/test/xpcshell
```

For release builds, see the [Tor Browser build documentation](https://gitlab.torproject.org/tpo/applications/tor-browser-build). `tools/fawkes/make-placeholder-icons.py` creates the placeholder graphics that tor-browser-build expects, and `tools/fawkes/check-branding.py` finds Tor Browser and Firefox names left in the UI.

## Documentation

The details of each part, with the hooks, the limits and the breakage risks, are in [`browser/components/fawkes/docs`](browser/components/fawkes/docs):

- [`architecture.md`](browser/components/fawkes/docs/architecture.md): components and processes
- [`categories.md`](browser/components/fawkes/docs/categories.md): all switches with defaults and breakage risks
- [`randomization.md`](browser/components/fawkes/docs/randomization.md): fingerprint randomization
- [`js-apis.md`](browser/components/fawkes/docs/js-apis.md) and [`css.md`](browser/components/fawkes/docs/css.md): JavaScript and CSS protection
- [`profiling.md`](browser/components/fawkes/docs/profiling.md): behavioral profiling protection
- [`network.md`](browser/components/fawkes/docs/network.md): referer, user agent, HTTP/3, ad blocker and local network
- [`proxy-chain.md`](browser/components/fawkes/docs/proxy-chain.md), [`proxy-scraper.md`](browser/components/fawkes/docs/proxy-scraper.md) and [`proxy-sources.md`](browser/components/fawkes/docs/proxy-sources.md): proxy chain and proxy scraper
- [`cookie-banners.md`](browser/components/fawkes/docs/cookie-banners.md): cookie banner handling
- [`tor-mode.md`](browser/components/fawkes/docs/tor-mode.md): running without Tor
- [`storage.md`](browser/components/fawkes/docs/storage.md): `fawkes-sites.json`

FawkesWebXE is based on [Tor Browser](https://gitlab.torproject.org/tpo/applications/tor-browser) and Mozilla Firefox, and uses the same licenses: see [`LICENSE`](LICENSE).
