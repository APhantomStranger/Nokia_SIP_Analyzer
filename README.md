# Nokia_SIP_Analyzer

# Nokia Sip Scope

A tool for digging through SBC / SIP logs without losing your mind. It's one HTML file — open it in a browser, drop a log in, and it pulls the SIP out, groups it into calls, and tries to tell you what went wrong.

Everything runs in your browser. Nothing gets uploaded anywhere — the file never leaves your machine — so you can throw production traces at it without worrying about where they end up.

## Getting started

Double-click `sipscope.html` (or open it in Chrome / Edge / Firefox / Safari, whatever you've got). You'll get a drop zone. Drag a log onto it, or click **Choose log file**. Any text log works — `.log`, `.txt`, `.siptrace`, whatever your SBC spits out.

Big files are fine. It reads the file in chunks so a multi-hundred-MB trace won't choke it, and you get a progress bar while it indexes.

Once it's done you land in the main three-pane view:

- **left** — the navigator (your list of calls / dialogs / events)
- **middle** — the message stream
- **right** — the inspector (click anything and it explains what it is)

## The format thing

It was tuned against real Nokia / Ribbon IMS traces — the ones with the `+++ 2020/10/08 …` record headers and the bracketed `[INVITE …]` / `[SIP/2.0 200 OK]` blocks. Those logs write the same wire message three or four times (pre-screened copy, TU copy, the actual send), so Sip Scope dedupes down to the real wire events using the `SENDING` / `RECEIVED` markers. Nice side effect: that's also where it grabs the real source/destination IPs and direction for each message.

It handles plainer SIP logs fine too, so it's not Nokia-only.

## The tabs (left pane)

There are four:

**Calls** — anything anchored by an INVITE. This is your call list. Each row is a Call-ID that has an INVITE in it. Click one to pull up the whole dialog.

**Other SIP** — the non-call stuff, grouped by Call-ID: REGISTERs, OPTIONS keepalives, SUBSCRIBE/NOTIFY, and so on. This is where you go when you're chasing registration or keepalive problems instead of calls.

**H.248 / Voice** — the media gateway control plane (H.248 / Megaco) plus RTP / media lines. Look here when the call sets up fine in SIP but there's no audio.

**Push** — push notification activity. On Nokia / IMS this lives under `IMS:NSI_CONN_MANAGER` (plus any APNs / FCM / PushKit lines). It's a little special, so there's a whole section on it below.

## Clicking around

Click any message in the middle stream and the right pane breaks it down — what the method or response code means, what it means *in SBC terms*, and what to check. Response codes, Q.850 cause codes, session timers, registration expiry, all of it.

Click a call in the left pane and it focuses just that call's messages. If the device that placed the call registered earlier in the log, Sip Scope links the two for you — the REGISTER shows up inline with the call, and there's a little "linked device registration" panel in the inspector. Handy for the "wait, was this phone even registered?" moments.

## Scan for Errors

The 🔎 button up top. This is the "just tell me what's broken" button. It sweeps the whole log and sorts every problem into buckets:

- Registration
- Call setup
- Media / Voice
- Push
- Gateway / H.248

Each finding is a card you can expand — it shows the evidence line, a plain-English explanation of what it means, and a list of SBC-specific things to check. Hit "go to" on a card and it jumps you straight to that call or log line. The chips across the top filter by bucket.

## Issues

The ⚠ button is the lighter-weight version — a quick roll-up of problem calls, problem dialogs, and error events, grouped by type. Click anything to jump to it. Think of **Scan** as the detailed view and **Issues** as the quick glance.

## Ladder diagrams

Focus a call, then hit the diagram button. You get a proper ladder / sequence diagram of the flow — who sent what to whom, in order. There's a **network** mode (real IPs / hosts) and a **conceptual** mode (caller / SBC / callee). You can zoom in, and export it as an SVG if you want to drop it in a ticket or email.

## About push (the interesting part)

Push failures are sneaky because the push usually *looks* like it worked. In both a good push log and a failed one, the connection manager logs `Succeeded to send a message to notification server` — the push went out fine at the SBC. Lines like `fail to find a … port … wait the query`, `err(0)`, and DNS TTL timer chatter look alarming but are completely normal, so Sip Scope doesn't flag them as errors.

The real failure shows up as a *consequence*, not an error line: the push went out, the device never woke up, and the INVITE to it times out with a 408. Sip Scope correlates the two and flags a **push wake-up failure**. A healthy push log, by contrast, shows the push going out and the call reaching 200 OK.

It knows both platforms now:

- **Google (FCM/GCM)** comes in two generations, and Sip Scope reads both:
  - the **older XMPP flow** — a stream to `fcm-xmpp.googleapis.com` / `gcm.googleapis.com` with JSON messages and `message_type: ack/nack` receipts;
  - the **newer HTTP v1 flow** — an HTTP/2 `POST /v1/projects/…/messages:send` to `fcm.googleapis.com` with an OAuth Bearer token. The SBC logs the decoded exchange (`sendNotification: the decoded sent HTTP message` / `handleOneHttpMsg`), and success is a `:status: 200` plus a `projects/…/messages/…` receipt in the response body — that receipt is the v1 equivalent of the old ack.

  For the **HTTP v1** flow specifically, Sip Scope now decodes the full error model from Nokia's FDD:

  - **The exact FCM error and whether it retries.** A push error names the code and status and says what the SBC does about it: **404 UNREGISTERED** (fatal, no retry — the device's token is dead, the client must register a new one), **403 SENDER_ID_MISMATCH** (fatal — wrong FCM project for that token, an app↔project mapping problem), **400 INVALID_ARGUMENT** (fatal — the SBC built a bad push: token format, payload >4096 bytes, or bad TTL), **429 QUOTA_EXCEEDED** (backoff — FCM throttling, usually affecting many subscribers at once), and **500 / 503 / timeout** (transient — the SBC retries once). A fatal code gets a different, more pointed diagnosis than "device asleep."
  - **The OAuth token stage.** Before it can POST, the SBC fetches a short-lived Bearer token from `oauth2.googleapis.com`. If that fails or there's no valid token, the SBC **skips the FCM POST entirely** — so Sip Scope flags "push NOT sent — no valid OAuth token," which is a credential problem (Service Account Private Key / connectivity) that breaks push for a whole app, not a per-device issue.
  - **Transport awareness.** It tells you whether a Google push used the old XMPP stream or the new HTTP v1 POST, and shows the FCM project id.
  - **PM counters & alarms glossary.** If a log or a pasted PM report mentions FCM counters (`VS.PCSCFFCMPOSTError`, `VS.FEPHFCMTokenError`, `VS.PCSCFFCMPOSTBlockQuota`, …) or alarms (`LSS_missingFCMAuthData`, `FCMPOSTReqErrRate`), the inspector decodes what each one measures and what a non-zero value means. `LSS_missingFCMAuthData` is a direct "your FCM credentials/CA/token aren't installed" red flag.
  - **Retry counter.** The payload's `"count"` field is read as the attempt number, so retries of one logical push aren't mistaken for separate failures.

  Either way the payload carries the target's **MDN** (phone number), the caller identity, and the P-CSCF the device should re-register to. Sip Scope pulls the MDN out and matches it to the INVITE, so the push↔call link is **exact**, not a guess. If the push service accepted the message (ack or v1 receipt) and the device still didn't answer, the problem is on the device (app killed, no data, battery optimization, stale token) — and the diagnosis says so.
- **Apple (APNs)** goes to `api.push.apple.com`; you'll see the `apns_topic` / `apns_push_type` decisions in the P-CSCF lines. The Apple payload isn't logged in the clear the way GCM's is, so Apple correlation stays time-based.

The Push tab shows sends, FCM acks, an Apple/Google breakdown, and genuine errors. Click a push line and the inspector tells you the platform and target; a call linked to a push gets a "Linked push notification" panel showing exactly where its push went out.

## Access vs core

The SBC has two faces: the **access side** (toward the device — TLS, the phone on a high ephemeral port) and the **core side** (toward the IMS core — well-known ports, internal addressing). Sip Scope classifies every wire message and tags it **ACC** or **CORE** in the stream, and the inspector shows which side plus the transport. A call's inspector also summarizes which legs it contains — and reminds you that if you only see one side, the mate leg is under a different Call-ID (the SBC rewrites Call-IDs between legs).

## The failure-cause database

Built from real failed-call traces, there's now a knowledge base of *why calls actually die* that goes beyond the response code:

- **Policy kills** — a 503 with `Reason: … "PO: AAA: … exp_result_code=5021"` isn't a capacity problem; it's the policy server (PCRF over Gq'/Rx) refusing to authorize media for the call. Sip Scope decodes the Diameter result code (5021/5065 = no bearer session, 5063 = service not authorized, 3004 = PCRF overloaded, and so on) and says so in plain English.
- **User not found** — 404s paired with `Q.850 cause 1 "User not Found"`, usually meaning the callee lost its registration or the number isn't provisioned.
- **Registration terminations** — reginfo NOTIFY bodies are parsed for the registration lifecycle (`registered` / `unregistered` / `expired` / `deactivated`). It's calibrated so the routine "old contact terminated, new contact active" full-state NOTIFYs in healthy logs don't false-alarm; only a genuine loss of the binding gets flagged.
- **Deliberate hangups** — a call that ends seconds after answer with `Reason: USER triggered` or Q.850 cause 16 is someone hanging up, not a media failure, and isn't flagged.

## Search and filters

There's a search box in the left pane — type a number, Call-ID, IP, method, response code, whatever, and it filters the list. There are also type filters (toggle 1xx / 2xx / 3xx / etc. on and off) and category toggles on the H.248 tab.

## Light / dark

Top-right button flips between the bright Nokia theme (the default) and dark mode, and it remembers which you picked. Dark mode is the deep-navy Nokia look — nice if you're staring at logs at 2am.

## Exporting

From a focused call you can export the call's messages, and export the ladder diagram as an SVG. Good for attaching to tickets or pasting into a writeup.

## A couple of honest caveats

- **Call-ID rewriting.** The SBC rewrites the Call-ID between the access leg and the core leg, so one end-to-end call can show up as two separate groups with different Call-IDs. Sip Scope links them by caller / callee number where it can, but if you're hunting one call and only see half of it, go look for the other leg under a different Call-ID.
- **Push correlation is by proximity, not a hard token.** The NSI lines don't carry the target number, so the push-to-call link is really a "push went out around this time and a call to a mobile timed out" inference. It's right for the common case, but it's a heuristic — keep that in mind before you blame the push gateway.
- **H.248 in these traces is light** — mostly "message from Gateway" pings rather than full transaction text. The full ITU-T H.248.8 error-code dictionary is built in, so if your gateway *does* log error codes they'll get decoded, but don't expect deep Megaco parsing if the codes aren't in the log to begin with.

## If something looks off

If it parses a weird format wrong, or your push / H.248 lines use markers it doesn't recognize, the parser is just regex under the hood and easy to extend — grab ~40 representative lines and they can be wired in.
