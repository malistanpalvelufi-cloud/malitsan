# Malistan Palvelu — website & quote-form backend

## SEO, favicon, and fixing the old "Index of /" Google result

This update added proper SEO metadata, a real favicon set (from the official
square Malistan "M" logo), and the files GitHub Pages needs to serve the
site correctly as `https://malistanpalvelu.fi/`.

**Files added at the repo root** (must stay at the root — not in a subfolder):
- `favicon.ico`, `favicon-32x32.png`, `favicon-192x192.png`, `apple-touch-icon.png`
  — generated directly from the uploaded square logo, resized only (never redrawn).
- `robots.txt` — allows normal crawling, references the sitemap.
- `sitemap.xml` — lists the single canonical homepage URL.
- `CNAME` — contains `malistanpalvelu.fi`, required for GitHub Pages to serve
  a custom apex domain instead of falling back to `username.github.io`.
- `assets/images/malistan-logo-square.png` and `assets/images/og-image.png`
  — used by the JSON-LD `logo`/`image` fields and the Open Graph/Twitter
  preview image.

**`index.html` `<head>` changes:**
- `<title>` and meta description updated to the exact requested copy.
- Added `<meta name="robots" content="index, follow">`.
- Added `<link rel="canonical" href="https://malistanpalvelu.fi/">`.
- Added the four favicon `<link>` tags referencing the files above by
  absolute root path (`/favicon.ico`, etc.) — correct as long as the site is
  served from the domain root, which it is on GitHub Pages with a CNAME.
- Open Graph, Twitter Card, and the JSON-LD `LocalBusiness` block were
  updated/cleaned up (see note below) and now point at the small logo/OG
  image files rather than any inline data.
- Nothing else in the file — layout, sections, colors, animations, contact
  form — was touched for this change.

**Bug fixed while doing this:** a few `<meta property="og:image">`,
`<meta name="twitter:image">`, and the JSON-LD `logo`/`image` fields had
accidentally been set to enormous inline base64 image data (multi-megabyte)
in an earlier pass, bloating the `<head>` to several megabytes. These are
now small `https://...png` URLs as they should be.

### D. Likely cause of the old "Index of /" result, and the fix

Google showing **"Index of /"** for a domain almost always means the actual
HTTP server at that URL is returning a raw directory listing instead of a
web page — that specific text is the standard output of Apache/Nginx
"autoindex" (or a generic static file server), not something GitHub Pages
itself produces. That points to one or more of these:

1. **DNS isn't (or wasn't) pointed at GitHub Pages.** Check `malistanpalvelu.fi`'s
   DNS records:
   - Apex domain (`malistanpalvelu.fi`): four `A` records pointing to
     `185.199.108.153`, `185.199.109.153`, `185.199.110.153`, `185.199.111.153`.
   - Or a `CNAME` record to `<username>.github.io` if using a `www` subdomain.
   - If DNS instead points at an old host (a plain file server, an unrelated
     static bucket, etc.), that old host is what's serving the directory
     listing — the fix is repointing DNS fully to GitHub Pages.
2. **`index.html` must be exactly named `index.html`, lowercase, at the root**
   of whatever branch/folder GitHub Pages is set to publish from (Settings →
   Pages → Source). If it's nested in a subfolder, or capitalized differently
   on a case-sensitive host, index resolution can fail.
3. **The `CNAME` file must be committed to the repo** (now included here).
   Setting a custom domain only through the GitHub UI can be overwritten by
   later deploys from workflows that don't preserve it; committing the file
   makes it durable.
4. **Enforce HTTPS** in Settings → Pages once the domain is verified, so
   Google doesn't see a mixed HTTP/HTTPS split result.

None of this can be fixed by anything in the HTML/JS itself — it's a
hosting/DNS configuration issue, which is why the instructions above are
about GitHub/DNS settings rather than code changes.



This update adds a secure backend for the existing quote/contact form.
The static site (`index.html`) is unchanged in design and continues to be
served from GitHub Pages exactly as before; only its form now submits to
a separate, secure serverless endpoint instead of doing nothing after
validation.

```
GitHub Pages (index.html)  --fetch()-->  Serverless endpoint (api/send-quote.js)  -->  Resend API  -->  malistanpalvelu.fi@gmail.com
```

The Resend API key never appears in `index.html`, in any frontend
JavaScript, or in this repository. It lives only as a server-side
environment variable on whatever platform hosts `api/send-quote.js`.

## What changed

- **New service**: "Kaivinkoneen kuljettaja" added to the Palvelut
  section and the quote form's project-type list, with its own visual
  (no existing service visuals were replaced).
- **New section**: "Palvelualueet" (Helsinki, Vantaa, Espoo) added
  between the existing Yrityksestä and Prosessi sections, using the
  site's existing glass/grid components — no new CSS or design system
  was introduced.
- **Quote form**: same fields, layout, and styling as before. Its
  submit handler now calls the secure backend via `fetch()`, with a
  loading state, success/error messages, a hidden honeypot field for
  basic spam protection, and a guard against duplicate submissions.
- **New backend**: `api/send-quote.js`, a small serverless function
  that validates the submission and sends it through Resend.

Nothing else in the layout, colors, typography, navigation, footer,
existing services, existing visuals, or animations was modified.

## 1. Deploy the backend

`api/send-quote.js` is written as a Vercel serverless function (uses the
global `fetch`, no npm dependencies). To deploy on Vercel:

1. Push this repository (or just the `api/` folder) to a Vercel project.
2. In the Vercel project's **Settings → Environment Variables**, add:
   - `RESEND_API_KEY` — your secret Resend API key
   - `RESEND_FROM` — a verified Resend sender, e.g.
     `Malistan Palvelu <tarjous@yourverifieddomain.fi>`
   - `ALLOWED_ORIGIN` — your live frontend origin, e.g.
     `https://malistanpalvelu.fi`
3. Deploy. Your endpoint will be available at
   `https://<your-vercel-project>.vercel.app/api/send-quote`.

The same handler works with minor adaptation on Netlify Functions or
Cloudflare Workers — swap the `module.exports = async function(req, res)`
signature for the platform's expected handler signature; the validation
and Resend-call logic stays the same.

**Important:** In Resend, you can only send from a domain you've verified
(SPF/DKIM records added). Do not invent or guess a sender address — use
whatever domain the business has actually verified in their Resend
account. Until a domain is verified, Resend's own sandbox sender can be
used for testing only, not production.

## 2. Point the frontend at your deployed backend

In `index.html`, the form's submit logic reads its target URL from:

```js
var QUOTE_API_URL = window.MALISTAN_QUOTE_API_URL || 'https://your-backend.example.com/api/send-quote';
```

Set it by adding one small `<script>` tag before the closing
`</body>` in `index.html` (or anywhere before the existing form script
runs), with your real deployed URL:

```html
<script>window.MALISTAN_QUOTE_API_URL = 'https://<your-vercel-project>.vercel.app/api/send-quote';</script>
```

This keeps the endpoint URL easy to change per environment without
touching the rest of the form logic. No secret ever goes in this value —
it's just the public URL of your own backend.

## 3. Test it

1. Deploy the backend and set the frontend URL as above.
2. Open the live site, fill in the quote form, and submit.
3. Confirm:
   - The button shows a loading state, then a success message.
   - An email arrives at `malistanpalvelu.fi@gmail.com` with all
     submitted fields and a timestamp.
   - Submitting with the browser's network tab open shows no API key
     anywhere in the request.
4. To test error handling, temporarily point `QUOTE_API_URL` at an
   invalid URL and confirm the form shows the error message instead of
   silently failing.

## Security notes

- `RESEND_API_KEY` must only ever be set as a server-side environment
  variable on the backend host. Never put it in `index.html`, any
  `<script>` tag, a public repo, or a GitHub Pages build.
- The backend validates required fields, checks email format, caps
  field lengths, and rejects/silently-drops submissions where the
  honeypot field is filled in.
- Resend error responses are logged server-side only; the client only
  ever receives a generic error message.
- Set `ALLOWED_ORIGIN` to your real frontend origin in production so the
  endpoint can't be called from arbitrary sites.

## Content accuracy

No fictional company details (projects, customers, testimonials,
certifications, machinery ownership, employee counts, years of
experience) were added anywhere in this update. The new "Kaivinkoneen
kuljettaja" service and "Palvelualueet" section only state the type of
work offered and the metropolitan areas served, per the verified
company information already in the project.
