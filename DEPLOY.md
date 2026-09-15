# Cutover: Wix → Cloudflare

**Order matters.** Wix currently serves the website *and* the DNS *and* holds the
domain registration. The MX records that deliver company email live on Wix's
nameservers. Cancelling Wix before DNS has moved takes down company email.

Do these in order. Don't skip ahead.

| | State today | State when done |
|---|---|---|
| Registrar | Wix.com Ltd. | Cloudflare Registrar |
| Nameservers | ns12/ns13.wixdns.net | Cloudflare |
| Website | Wix (185.230.63.x) | Cloudflare Pages |
| Email | Google Workspace (MX → aspmx.l.google.com) | unchanged |

Domain expires **2026-11-20**. Transferring adds a year, so do step 5 before then.

---

## 1. Record what Wix is serving today

Before touching anything, dump the current DNS so you can diff against it later:

    dig evenstarnav.com ANY +noall +answer
    dig evenstarnav.com MX +short
    dig evenstarnav.com TXT +short
    dig www.evenstarnav.com +short

Save that output. Also open the Wix DNS panel and screenshot every record —
`dig` only shows what's queried, not the full zone. Pay attention to:

- **MX** — five `aspmx.l.google.com` entries (priorities 10/20/30/40/50)
- **TXT** — Google site verification, and SPF (`v=spf1 include:_spf.google.com ~all`)
- **DKIM** — usually `google._domainkey` as a TXT or CNAME
- **DMARC** — `_dmarc` TXT, if set
- any subdomains in use

## 2. Create the company Cloudflare account and import the zone

1. Sign up at cloudflare.com with a **company address** — ideally a shared mailbox
   (`admin@evenstarnav.com`), not a personal one. This account will hold the domain;
   it should outlive any one employee.
2. Add site → `evenstarnav.com` → Free plan.
3. Cloudflare scans and imports the existing records.
4. **Compare the imported zone against your step 1 notes, record by record.** The
   scan misses records it cannot enumerate. Add anything missing by hand.
5. Confirm MX, SPF, DKIM and DMARC are all present and exactly match. This is the
   step that protects company email — do not rush it.

Do **not** change nameservers yet.

## 3. Switch nameservers

In the Wix domain panel, set the nameservers to the two Cloudflare gives you.

Propagation is usually minutes, up to 24h. Verify:

    dig evenstarnav.com NS +short        # expect the Cloudflare pair
    dig evenstarnav.com MX +short        # expect the five Google entries, unchanged

**Send a test email to and from a company address before continuing.**

At this point the website is still on Wix — Cloudflare is just serving the same
records. Nothing visible has changed.

## 4. Deploy the site and point the domain at it

1. Cloudflare dashboard → Workers & Pages → Create → Pages → connect to Git.
2. Select `Evenstar-Navigation/landing-page`, branch `main`.
3. Build command: **leave empty**. Build output directory: **`/`**.
   There is no build step — the repo is the site.
4. Deploy, then open the `*.pages.dev` URL and check both pages and the email link.
5. Custom domains → add `evenstarnav.com` and `www.evenstarnav.com`.
   Cloudflare creates the records and issues the certificate automatically.
6. Verify:

       curl -sI https://www.evenstarnav.com | head -1
       dig www.evenstarnav.com +short

The site is now off Wix. Email is untouched throughout.

## 5. Transfer the registration to Cloudflare

Only after steps 3 and 4 are confirmed working.

1. In Wix: unlock the domain and request the **authorization code** (EPP code).
   Confirm WHOIS privacy is off if it blocks the transfer.
2. Cloudflare → Domain Registration → Transfer Domains → enter the code.
3. Approve the confirmation email. Transfer takes up to 5 days; the site and email
   keep working the whole time because the nameservers are already Cloudflare's.

## 6. Cancel Wix

Last. Not before.

- Confirm the registration now shows Cloudflare: `whois evenstarnav.com | grep -i registrar`
- Download anything still only in the Wix media library.
- Cancel the Wix premium plan and any Wix domain auto-renewal.

---

## Rollback

Through step 4 the escape hatch is the same: in Cloudflare DNS, point the apex and
`www` records back at the Wix IPs from your step 1 notes. Takes effect in minutes.

After step 5 the registrar has moved and rollback means transferring back — so treat
step 5 as the point of no return, and only do it once the site has been live and
correct for a few days.

## Checklist

- [ ] 1. Existing DNS dumped and screenshotted
- [ ] 2. Company Cloudflare account created (shared mailbox, not personal)
- [ ] 2. Imported zone diffed against step 1 — MX, SPF, DKIM, DMARC all verified
- [ ] 3. Nameservers switched; MX verified; test email sent and received
- [ ] 4. Pages project deployed; `*.pages.dev` checked
- [ ] 4. Custom domains added; https live on apex and www
- [ ] 5. Registrar transferred to Cloudflare
- [ ] 6. Wix media downloaded; Wix plan and domain auto-renew cancelled
