---
title: "Intigriti June Bonus Challenge write-up - There's a Leak in the Jar-O"
tags: [intigriti, csrf, ctf, web]
---

# There's a Leak in the Jar-O

**Author:** zzzquil  
**Challenge:** Intigriti June Bonus Challenge - LeakyJar  
**Category:** CSRF  
**Status:** ✅ Solved  

---

## The Target

LeakyJar is a small Flask app where bakers store recipes in a private "recipe box" and can **share the whole box** with another baker by username. The `/submit` endpoint (the "Report a recipe" page) hands a URL to the **Master Baker** - an admin bot that visits it. The flag lives in the Master Baker's box.

The setup reads like a stored-XSS challenge: get JavaScript running in the bot's session and exfiltrate something. So that's where I started.

---

## Step 1 - Every reflected surface is escaped

I put a marker into every field a victim renders and read the raw HTML back:

| Surface | Output |
|---|---|
| Review name / text | `&lt;img&gt;` - escaped |
| Search `?q=` | escaped, inside a quoted attribute |
| Vault recipe name / notes | escaped (even in the shared view) |
| Username in `<title>` / `<h1>` | `zz&#34;&gt;&lt;b&gt;` - escaped |

`{{7*7}}` also came back as `{{7*7}}`, ruling out SSTI. And the responses ship **no CSP at all** - which would be a gift, except there's nothing to pair it with: Jinja2 autoescapes every sink. The recipe page is a museum of other people's dead `<img onerror>` and `<svg onload>` attempts.

No reflection means no XSS. That's not a dead end, it's a signpost: if the bug isn't *in* the page, it's in what the bot can be made to *do*.

---

## Step 2 - Fingerprinting the bot

I pointed `/submit` at a webhook to see what shows up:

```
sec-ch-ua: ...HeadlessChrome
sec-fetch-site: none
sec-fetch-mode: navigate
sec-fetch-dest: document
```

A real headless Chrome, doing a **top-level navigation** to an **external** URL. So it runs JavaScript, carries its cookies, and will browse to any page I host. The admin will walk into my page logged in.

---

## Step 3 - The share form has no lock on the door

The *Share my recipe box* form:

```html
<form method="POST" action="/share">
  <input name="username">
</form>
```

No CSRF token. It shares the **entire box** with whatever username you submit. And the session cookie:

```
session=...  SameSite=None; Secure; HttpOnly
```

`SameSite=None` is the whole ballgame: a cross-site `POST` to `/share` **still carries the cookie**. The cookie attributes are set by the server, so the bot gets the same ones - and the fact that the attack lands is itself proof the bot's cookie rode along. No XSS required; the intended bug is a plain CSRF.

(`/share` is POST-only - GET returns *Method Not Allowed* - so it has to be a form submit, not a link.)

---

## Step 4 - Stand and deliver

One file, on any origin I control:

```html
<form id="f" action="https://leakyjar.intigriti.io/share" method="POST">
  <input name="username" value="zzzquil">
</form>
<script>document.getElementById('f').submit()</script>
```

1. Host the page.
2. Submit its URL via `/submit`.
3. The Master Baker opens it → his browser POSTs to `/share` in his session → he shares his box with `zzzquil`.
4. Reload `/vault` as `zzzquil` → **Shared with you** → open the admin's box.

Sitting between *Brown Butter Base* and a couple of recipes I'd added uninvited:

> **Master Baker's Secret Recipe** - `INTIGRITI{XXXXXXFLAGXXXXXX}`

---

## The Fix

- Add a per-request anti-CSRF token to `/share` and every state-changing POST, validated server-side.
- Set the session cookie to `SameSite=Lax` or `Strict`.
- Validate `Origin`/`Referer` on state-changing requests.

---

## Closing thoughts

The challenge baits you toward XSS - a no-CSP page covered in user-controlled fields - and then escapes every last one of them. Burning each surface isn't wasted work; it's what points at the real class of bug. A share form with no token and a cookie set to `SameSite=None` was the leak the whole time. The jar was never going to spill from a script tag.

---

*Write-up by zzzquil - Intigriti June Bonus Challenge, June 2026*
