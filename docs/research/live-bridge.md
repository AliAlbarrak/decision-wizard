# Live bridge between Claude Code and the Decision Wizard

Research date: 2026-11-11. Repo state: branch `docs/live_bridge_research`, wizard `index.html` as checked out.

## Question

Which web technologies could, as an optional fast path, let Claude Code (a CLI agent on the user's Mac) push a round into the Decision Wizard page at once and get the user's answers back without writing and reading round files? File exchange through `wizard-rounds/` stays the default and the fallback.

## Short answer

1. **Claude → page is already close to instant.** The page watches the folder with `FileSystemObserver` (Chrome 133 and later) and also polls every 4 s. A new round file opens within about a second. The gap is in the other direction: Claude only learns that answers exist when the user tells it.
2. **Close that gap first with the Claude Code `Monitor` tool, which needs no change to the page.** The skill starts a background watch on the round file, and Claude reacts when the answers land. This still uses files, but it removes the "tell Claude you're done" step in every browser that has folder access. Effort: S.
3. **For a true no-file path, use a loopback helper on `127.0.0.1` with a token, for Chrome and Edge only.** The skill runs the helper itself, either under `Monitor` or as a plugin MCP server. The page connects with `fetch(..., { targetAddressSpace: "loopback" })` plus SSE or WebSocket. Chrome shows a one-time Local Network Access prompt for loopback, and the user must allow it. Safari blocks `https` → `http://localhost` as mixed content, so Safari keeps the file path. Effort: M.
4. **Skip native messaging, WebRTC and WebTransport.** Claude Code *channels* fit well technically but are a research preview: custom channels need `--dangerously-load-development-channels`, so they can't be the shipped default yet.

## Current mechanisms in the app

| Mechanism | Where | How it works |
|---|---|---|
| Folder connection (File System Access) | `index.html:4002` `fsSupported()` checks `showDirectoryPicker`; `index.html:4760` opens the picker; `index.html:4018-4025` query/request `readwrite` permission | The user picks the project's `wizard-rounds/` folder once. The handle is stored, and permission is re-queried on later visits. |
| Watcher election | `index.html:5454-5455` (`navigator.locks.request("decision-wizard-watch", …)`), `index.html:5409` `becomeWatcher()` | Only one open wizard page watches. The others follow it through `BroadcastChannel("decision-wizard")` (`index.html:5421-5430`) and `storage` events. |
| Folder watch | `index.html:4171-4181` `startWatch()` uses `new FileSystemObserver(cb).observe(dirHandle)` when available, then `scanSoon()` after `TIMING.scanDelay` = 150 ms. `index.html:5417` polls every `TIMING.poll` = 4000 ms (`index.html:6155`), with a light scan 4 times out of 5 (`index.html:4170`). It also scans again on focus or visibility. | New round files open automatically. A scan only reads. |
| Answer write-back | `index.html:4663-4700` `writeLinked()` writes with `getFileHandle(name,{create:true})` → `createWritable()`, and aborts if the file changed since it was read. `index.html:9810` `fileForm()` | Every answer is saved into the round file as `{ meta, answers, round }`. `meta` carries `total`, `answered` and `open` counts (`d-wizard/SKILL.md:166-169`). The file has no explicit "finished" flag. |
| Claude reads answers | `d-wizard/SKILL.md:157-162` | Claude rereads the file *when the user says they have answered*. Nothing tells Claude on its own. |
| `#open=` link | `index.html:5461-5464`, `5481-5490` `sentFrom()`, `5508` `openSent()` | The URL fragment carries the whole round as base64url JSON `{file, round, answers}`. `hashchange` and initial load are handled in `setupSent()` (`index.html:5565-5579`). |
| `web+dwizard:` protocol | `manifest.webmanifest` `protocol_handlers: [{protocol:"web+dwizard", url:"./?open=%s"}]`; `index.html:5465-5468`, `5526` `openRef()` | The link carries only `{file, title}`, a reference. The page then finds the file in a connected folder. The in-code comment says the OS takes "only about 2,000 characters", which this research did not verify. |
| Focus existing window | `manifest.webmanifest` `launch_handler: {client_mode: "focus-existing"}`; `index.html:5567-5568` `launchQueue.setConsumer(lp => sentArrive(lp.targetURL))` | A launch reaches the already-open app window through the launch queue, and is de-duplicated within 4 s (`index.html:5496`). |
| Clipboard out | `index.html:3210-3223`, `9157-9173` | The page copies the answers JSON to the clipboard. |
| Service worker | `sw.js` | It handles only same-origin GETs and Google Fonts. A cross-origin request, such as one to `127.0.0.1`, passes through untouched. |
| Opening the wizard | `d-wizard/SKILL.md:151` | Claude Code runs `open https://abdurrahmankh.github.io/decision-wizard/` once per session. |

What this means: the **push** half already exists twice over, as files plus the observer and as `#open=` / `web+dwizard:`. The **return** half has no signal at all.

---

## 1. Loopback HTTP helper (fetch / SSE / WebSocket / WebTransport)

**How it would work**
- **Helper:** a tiny process that Claude Code starts, bound to `127.0.0.1:<random port>`, holding a random token. It could be run under `Monitor` (§5b), shipped as a plugin `bin/` tool, or embedded in an MCP server (§5).
- **Push:** Claude hands the round to the helper on stdin or argv. The page receives it over an `EventSource`/WebSocket stream from the helper, or by polling `GET /round`.
- **Answers:** the page `POST`s the answers JSON to `/answers`, on every change or on finish. The helper prints it to stdout (picked up by `Monitor`) or returns it from a tool call (MCP).
- **Bootstrap (port + token to the page):** Claude runs `open "https://abdurrahmankh.github.io/decision-wizard/#bridge=<port>.<token>"`. The hash never reaches GitHub's servers. Alternatively it runs `open "web+dwizard:bridge/<port>.<token>"`, which reaches the installed app through the protocol handler and launch queue (§2).

**Can the https github.io page reach `http://127.0.0.1`?**

| Browser (macOS) | Mixed content | Permission | Result |
|---|---|---|---|
| Chrome | Loopback counts as potentially trustworthy, so it isn't blocked as mixed content (WICG LNA §2.5: "we do not need special handling for the loopback case as it is already considered to be potentially trustworthy"). `ws://` to `localhost` or a loopback literal is exempt as well (Chrome LNA guide). | **Local Network Access.** The prompt for fetch/XHR launched in Chrome 142 (Chrome blog, update 2025-09-29). WebSockets were gated from Chrome 147 (release notes, 2026-04-07), and WebTransport in 147 too. The permission is now split into `local-network` and `loopback-network`, and a public → loopback request needs `loopback-network`. It works only in secure contexts. | **Works after a one-time user prompt.** A denied prompt means the request fails. Query `navigator.permissions.query({name:"loopback-network"})` and treat a `TypeError` as `"prompt"`. Don't query `local-network-access`, which crashes older renderers (Chrome LNA guide). |
| Edge | Chromium-based, so presumably the same as Chrome. **Unverified:** no Microsoft source was checked for Edge's LNA rollout or prompt. | Same as Chrome, unverified. | Probably works the same. Needs a quick test. |
| Safari | **Blocked.** WebKit bug 171934, "Don't treat loopback addresses … as mixed content", is still **NEW** and depends on 250607 "Implement Local Network Access" (also NEW). A comment dated 2026-08-19 reads: "All vendors except Safari now ship LNA in stable." WebKit PR #77520, adding a built-in LNA prompt, is an open draft from Oct 2026 and not shipped. | n/a | **Not possible** from the https page today. Keep files. |

- **Installed PWA:** LNA permissions are per origin, so the installed app and a tab share the github.io origin. Nothing in the sources suggests app windows are treated differently (unverified). `sw.js` doesn't intercept the request.
- **`file://` local copy:** behavior is **unverified**. Such a page has an opaque origin (`Origin: null`), and how Chrome classifies a `file:` initiator's address space wasn't confirmed. Treat it as unsupported for the bridge.
- **CORS:** the helper must answer `Access-Control-Allow-Origin: https://abdurrahmankh.github.io`, never `*`. It must handle the `OPTIONS` preflight for `POST` with a JSON content type. The old Private Network Access preflight header (`Access-Control-Allow-Private-Network`) is no longer needed: Chrome's guide says not to add PNA handling to new code.
- **Transports:**
  - `fetch` + `EventSource` (SSE) is simplest. Chrome documents `targetAddressSpace: "loopback"` for `fetch`/`Request`, which also fails the request if the address resolves elsewhere.
  - WebSocket works too, because `ws://localhost` is exempt from mixed content. `targetAddressSpace` for WebSocket only arrived in Chrome 154 and isn't needed for a loopback literal.
  - WebTransport needs `https://` with a certificate, so it's needless effort here.

**Security**
- **Token:** any website the user has granted loopback permission to, or any local process, can reach the port. Require the random per-session token, in a header or the first message, on every request.
- **Origin check:** check `Origin` against the github.io origin and reject everything else.
- **Binding and DNS rebinding:** bind only `127.0.0.1`. Check `Host` is `127.0.0.1:<port>` to defeat DNS rebinding.
- **Payload:** treat the answers as untrusted input to Claude, since they are prompt-injection-capable text. Claude Code's channel docs make the same point: "An ungated channel is a prompt injection vector."

**Effort:** M. That covers a ~100-line helper (Node or Python stdlib), page code for connect, permission and stream with a fallback to files, and SKILL.md steps.

**Verdict:** **Recommended as the optional fast path, Chrome and Edge only.** Detect it and fall back to files silently.

## 2. URL / protocol-handler push (`#open=`, `web+dwizard:`)

**Push**
- **`#open=` link:** carries the whole round. Chromium caps URLs at 2,097,152 characters (`url::mojom::kMaxURLChars`), so in-browser size isn't the issue.
  - The `open` command hands the URL to macOS Launch Services and then to the default browser. The OS-side length limit is **unverified**; the code comment's "about 2,000 characters" applies to the protocol path and wasn't confirmed either.
  - Chrome 139 made link capturing into installed PWAs the default on Windows, Mac and Linux, but Chrome's docs describe captured *clicks in pages*. Whether a link opened from the OS (`open https://…`) is captured into the app window is **not documented**, so a prototype is needed.
- **`web+dwizard:` link:** the manifest `protocol_handlers` member has been in Chrome since 96. A registered PWA opens when a link is clicked "from a browser or a platform-specific app", and the first protocol launch shows a consent dialog with the app's name and origin. With `focus-existing`, the launch goes to the existing window, and the page's `launchQueue` consumer receives `targetURL`. MDN marks `protocol_handlers` experimental, and it isn't Baseline.
  - Safari (macOS "Add to Dock" apps) support is **unverified**. MDN's compatibility table wasn't readable in this research.
- Either link can also carry a bridge bootstrap (`#bridge=` or `web+dwizard:bridge/…`) instead of a round. That fixes the length limit and keeps rounds out of URLs.

**Answers (return path):** URLs only go one way, into the page. There are three ways back:
- (a) **Clipboard:** already implemented (`index.html:3210`). Claude can read it with `pbpaste`, but the user still has to say "done", and reading the clipboard unasked is intrusive.
- (b) **Loopback endpoint:** POST to the helper (§1).
- (c) **Custom scheme owned by a native app:** the page navigates to `x-claude-wizard://…`. This needs a native app registered with Launch Services, and Chrome shows an "Open app?" prompt. That's an external-protocol dialog in Chrome UI, **unverified** as to wording. Effort is L, and it isn't worth it.

**Effort:** S for push, which exists already. **Verdict:** keep it as is. Use it to *bootstrap* the §1 bridge and to bring the window to the front. It isn't a return channel.

## 3. FileSystemObserver (still files)

- **Status:**
  - Origin trial ran Chrome 129–134.
  - It shipped in Chrome 133 stable (2025-02-04): "The `FileSystemObserver` interface notifies websites of changes to the file system." It covers user-granted local files and directories and OPFS.
  - MDN still marks it experimental and non-standard. Its spec PR is whatwg/fs#165.
- **Use in the app:** the app already uses it (`index.html:4176-4178`), with the 4 s poll as backup. So on Chrome and Edge the push side is already event-driven.
- **Limits:** it's page-side only. It doesn't help Claude notice answers.

**Effort:** none. **Verdict:** already done. Still files.

## 4. Browser extension + Native Messaging

**How it would work:** the page talks to an extension, through `externally_connectable` `matches` plus `chrome.runtime.sendMessage` from the page, or through a content script. The extension calls `runtime.connectNative()` to reach a native host, and Claude Code talks to that host through a socket or file.

**Costs**
- **Install:** the user has to install an extension, published to the store or sideloaded.
- **Host manifest:** the native host needs a manifest at `~/Library/Application Support/Google/Chrome/NativeMessagingHosts/<name>.json`, with `allowed_origins` listing the extension ID and no wildcards. Edge uses its own path, which wasn't checked.
- **Message limits:** 1 MB host → extension and 64 MiB extension → host. Framing is a 32-bit length prefix plus JSON over stdio.
- **Host lifetime:** a `connectNative` port keeps the host alive while the port is open.
- **Safari:** Safari needs a Safari Web Extension packaged in a macOS app, which wasn't researched in detail here.

**Security:** strong, because only the listed extension can launch the host. But the trust surface is large: an extension with host permissions.

**Effort:** L. **Verdict:** no. It's heavy for an optional nicety, and §1 gives the same result in Chrome and Edge with no install.

## 5. MCP server shipped with the skill (stdio to Claude Code + HTTP to the page)

**How it would work:** turn d-wizard into a plugin that bundles an MCP server.
- A plugin's `.mcp.json` starts it automatically, using `${CLAUDE_PLUGIN_ROOT}` paths.
- The server speaks stdio to Claude Code and also opens the §1 loopback HTTP listener for the page.
- Tool `ask_round(round)` → the server bootstraps the page (`open` with `#bridge=`) and blocks until answers arrive.

**Long-running tool calls in Claude Code** (code.claude.com/docs/en/mcp):
- **Timeouts:** stdio servers have no per-request timer. `MCP_TOOL_TIMEOUT` defaults to "about 28 hours", or set a per-server `timeout` in `.mcp.json`.
- **Idle timeout:** 30 minutes for stdio servers. A call with no response and no progress notification for that long aborts. Progress notifications reset the idle window, so the server should send progress every few minutes while the user answers. `CLAUDE_CODE_MCP_TOOL_IDLE_TIMEOUT` changes it.
- **Auto-backgrounding:** a call still running after 2 minutes "moves to a background task instead of blocking the session" (`CLAUDE_CODE_MCP_AUTO_BACKGROUND_MS`). That suits a user who takes 10 minutes to answer. **Unverified:** exactly how Claude is told when a backgrounded MCP call completes. Prototype it.
- **MCP tasks:** an alternative, public beta in Claude Code v2.1.304+. The call returns at once with a `taskId`, and Claude Code polls `tasks/get`.
- **Output limits:** results over 10k tokens warn, the default cap is 25k tokens, and text over 50k characters is saved to a file. Answers JSON is usually far smaller.

**5a. Channels: push into the session, no blocking call**
- An MCP server that declares `capabilities.experimental['claude/channel']` can emit `notifications/claude/channel { content, meta }`. Claude sees it as `<channel source="…">` and reacts in the running session. A reply tool makes it two-way.
- The official webhook example is exactly this shape: stdio MCP plus `Bun.serve` on `127.0.0.1:8788`, with SSE for outbound.
- **Blockers:**
  - Research preview.
  - The user must start Claude with `--channels`.
  - Custom channels aren't on the allowlist and need `--dangerously-load-development-channels`, which shows a full-screen warning.
  - Team and Enterprise orgs must set `channelsEnabled`.
  - Requires claude.ai or Console auth, and isn't available on Bedrock, Vertex or Foundry.
  - On the v2 MCP runtime, a server that negotiates protocol 2026-07-28 isn't registered as a channel; use `MCP_PROTOCOL_NEGOTIATION=legacy`.
  - Events drop silently when the server isn't loaded as a channel.

**5b. `Monitor` tool: the lightest "push into session"**
- Claude Code's built-in `Monitor` "runs a command in the background and feeds each output line back to Claude". It can also open a WebSocket and treat each text frame as an event; frames over 1 MiB end the watch.
- **Deadline:** 5 minutes by default, at most 30 minutes (10 in `-p` runs). Claude gets one notice at the deadline and can re-arm the watch.
- **Permissions:** Monitor uses the Bash permission rules.
- **Plugin monitors:** plugins can declare monitors that start with the session, or when a named skill first runs (`when`). Those run outside the sandbox and not in `-p` mode.
- **No-change form:** the skill tells Claude, after writing a round, to start `Monitor` with a small script. The script waits until the round file has `meta` and `meta.open == 0`, or a "finished" marker, then prints one line. Claude then reads the file and acts. Still files, but no user prompt is needed. Works wherever the folder connection works.
- **Bridge form:** `Monitor` runs the §1 helper itself. The helper prints the answers as one JSON line, so no MCP server is needed.

**Hooks:** a `FileChanged` hook exists, but per the hooks reference its output "shows stderr to user only" and can't inject context. `asyncRewake` can wake Claude from a background hook on exit code 2. That's possible, but `Monitor` is the documented tool for this.

**MCP elicitation, URL mode (spec 2025-11-25):** the server could ask the client to open the wizard URL. The client must show the full URL and get consent, and the client isn't told the outcome. It adds a consent click and nothing over `open`. Claude Code's support for URL mode wasn't confirmed.

**Effort:** MCP plus blocking tool: M–L, because of plugin packaging and a Node or Bun runtime. Channels: M, but preview-gated. Monitor watch: S.
**Verdict:** start with **5b**. Consider the MCP `ask_round` tool when d-wizard ships as a plugin anyway. Revisit channels when custom channels can be allowlisted without the dangerous flag.

## 6. Other channels (one line each)

- **BroadcastChannel / postMessage:** same-origin browsing contexts only. The CLI isn't one. Irrelevant, though the app already uses it between its own tabs (`index.html:5421`).
- **WebRTC data channel:** needs signaling plus a native WebRTC stack in the helper. Also gated by LNA on Chrome (crbug 421223919 listed it as planned). Too heavy.
- **Web Bluetooth / Web Serial / WebUSB:** not applicable.
- **Async Clipboard API:** the page can write the clipboard on a user gesture, which it already does, and Claude can `pbpaste`. That's a manual return path and needs a "done" signal. Fine as a last-resort fallback only.
- **Claude Code hooks:** see §5. `FileChanged` can't feed Claude, and `Monitor` is the right tool.
- **Remote Control / cloud sessions:** they drive the session from claude.ai and are unrelated to the page.

## Comparison

| Option | Push (Claude → page) | Answers (page → Claude) | Chrome / Edge (macOS) | Safari (macOS) | User prompts | Still files? | Effort | Verdict |
|---|---|---|---|---|---|---|---|---|
| Today: folder + FileSystemObserver + poll | ~instant (observer) / ≤4 s | Saved to file. Claude reads only when told | Yes | No folder access (no `showDirectoryPicker`) | Folder picker once, re-grant per visit in a tab | Yes | done | Keep as default |
| **Monitor watch on round file (5b)** | as today | Claude notified when answers land | Yes | n/a (no folder) | Claude Code may ask to run the watch | Yes | **S** | **Do first** |
| **Loopback helper + token (1)**, run via Monitor or MCP | Instant (SSE/WS) | Instant (POST) | Yes, after the LNA loopback prompt (Chrome 142+ fetch, 147+ WS). Edge unverified | **Blocked** (mixed content, WebKit 171934) | LNA prompt once per origin; Claude Code permission for the helper | No | **M** | **Optional fast path** |
| MCP `ask_round` blocking tool (5) | Instant | Tool result | Yes (same LNA prompt) | Blocked | MCP server consent; LNA | No | M–L | Later, if packaged as a plugin |
| Channels (5a) | via page/helper | Pushed into session | Yes | Blocked | `--dangerously-load-development-channels` warning; org setting | No | M | Not yet (preview) |
| `#open=` / `web+dwizard:` (2) | Yes (exists) | No | Yes; OS→app capture unverified | `#open=` in tab: yes. Protocol handler: unverified | Protocol consent on first launch | No (push only) | done | Use for bootstrap and focus |
| Extension + Native Messaging (4) | Yes | Yes | Yes, with extension install | Needs app-wrapped extension | Extension install | No | L | No |
| Clipboard (6) | n/a | Manual (`pbpaste`) | Yes | Yes | User gesture | No | done | Fallback only |
| WebRTC / WebTransport (6) | Possible | Possible | Gated by LNA; needs cert or signaling | Unknown | LNA | No | L | No |

## Open questions / prototype checklist

1. **"Finished" signal.** Answers are saved on every change (`writeLinked`). A watcher needs to know when the user is *done*. Is `meta.open == 0` enough, or should the page write `meta.finished_at` when the user finishes (`finishRound()`, `index.html:6891`)? This is a small page change with a big effect on 5b.
2. **Chrome LNA in the installed PWA.** Confirm the loopback prompt appears in the app window, check its wording (secondary sources call the site-setting "Apps on device", **unverified**), and confirm the grant persists across app launches.
3. **Edge.** Confirm LNA loopback gating and prompt on current stable Edge for macOS. No Microsoft source was checked.
4. **`open` from the CLI into the installed app.** Does `open "https://…/decision-wizard/#bridge=…"` land in the PWA window through link capturing and `launchQueue`, or in a browser tab? Does `open "web+dwizard:…"` reach the app with `focus-existing`? What is the macOS length limit on either?
5. **Backgrounded MCP calls.** When an `ask_round` call auto-backgrounds after 2 min, how does Claude get the result, and does it wake the session?
6. **Monitor permissions.** Will an auto-mode classifier or user permission prompt fire for each watch, and can the skill rely on a narrow `Bash(...)` allow rule?
7. **Safari.** Recheck WebKit bug 171934, bug 250607 and PR #77520 before each release. If WebKit ships LNA, the §1 path may open up for Safari with a prompt.
8. **`file://` copy.** Test whether a `file://` page can reach the helper (Origin `null`) in Chrome. If it can, allow `null` only with the token.

## Sources

- WICG, Local Network Access (Draft CG Report, 7 Aug 2026): https://wicg.github.io/local-network-access/
- Chrome for Developers, New permission prompt for Local Network Access (2025-06-09, updated 2025-09-29): https://developer.chrome.com/blog/local-network-access
- Chrome for Developers, Local Network Access guide: https://developer.chrome.com/docs/modern-web-guidance/guides/security/local-network-access
- Chrome 147 release notes (LNA for WebSockets and WebTransport, 2026-04-07): https://developer.chrome.com/release-notes/147
- Chrome 154 release notes (`targetAddressSpace` for WebSockets, 2026-09-22): https://developer.chrome.com/release-notes/154
- MDN, Local network access: https://developer.mozilla.org/en-US/docs/Web/Security/Defenses/Local_network_access
- MDN, Mixed content (loopback treated as secure origin): https://developer.mozilla.org/en-US/docs/Web/Security/Defenses/Mixed_content
- WebKit bug 171934 (loopback as mixed content, status NEW): https://bugs.webkit.org/show_bug.cgi?id=171934
- WebKit PR #77520 (LNA built-in prompt, draft): https://github.com/WebKit/WebKit/pull/77520
- Chromium `url::mojom::kMaxURLChars` = 2097152: https://chromium.googlesource.com/chromium/src/+/main/url/url_constants.h
- Chrome for Developers, URL protocol handler registration for PWAs (Chrome 96): https://developer.chrome.com/docs/web-platform/best-practices/url-protocol-handler
- Chrome for Developers, Launch Handler API: https://developer.chrome.com/docs/web-platform/launch-handler
- Chrome for Developers, Navigation management into installed PWAs (Chrome 139 link capturing): https://developer.chrome.com/docs/capabilities/pwa-navigation-management
- WHATWG HTML, custom scheme handlers: https://html.spec.whatwg.org/multipage/system-state.html#custom-handlers
- MDN, manifest `protocol_handlers`: https://developer.mozilla.org/en-US/docs/Web/Manifest/Reference/protocol_handlers
- Chrome 133 release notes (FileSystemObserver, 2025-02-04): https://developer.chrome.com/release-notes/133
- Chrome for Developers, File System Observer origin trial: https://developer.chrome.com/blog/file-system-observer
- MDN, FileSystemObserver: https://developer.mozilla.org/en-US/docs/Web/API/FileSystemObserver
- Chrome for Developers, Native messaging: https://developer.chrome.com/docs/extensions/develop/concepts/native-messaging
- Chrome for Developers, `externally_connectable`: https://developer.chrome.com/docs/extensions/reference/manifest/externally-connectable
- Claude Code, MCP (timeouts, auto-backgrounding, tasks, channels note): https://code.claude.com/docs/en/mcp
- Claude Code, Channels: https://code.claude.com/docs/en/channels
- Claude Code, Channels reference: https://code.claude.com/docs/en/channels-reference
- Claude Code, Tools reference (Monitor): https://code.claude.com/docs/en/tools-reference
- Claude Code, Plugin components (monitors, MCP servers, channels): https://code.claude.com/docs/en/plugins/components
- Claude Code, Hooks reference (`FileChanged`, `asyncRewake`): https://code.claude.com/docs/en/hooks
- MCP spec (draft), Elicitation (URL mode since 2025-11-25): https://modelcontextprotocol.io/specification/draft/client/elicitation
