# Cutover: Wix → GitHub Pages

**This replaces an earlier Cloudflare-based plan.** That plan required moving the
domain's nameservers away from Wix, which turned out to be blocked: Wix (like
Shopify and other website builders) does not let you repoint nameservers while it
remains the registrar — confirmed both in the Wix DNS panel ("NS records are not
editable") and in [Cloudflare's own docs on transferring from website
builders](https://developers.cloudflare.com/registrar/get-started/transfer-domain-to-cloudflare/#transfer-from-website-builders).
The only route to Cloudflare owning DNS is: transfer the domain to an intermediate
registrar, wait the ICANN-mandated 60 days, then transfer again to Cloudflare — and
that transfer also unavoidably exposes registrant WHOIS info at Wix, since Wix
bundles privacy protection and DNSSEC into a single toggle.

None of that is worth it for the actual goal, which is dropping the Wix hosting
subscription. GitHub Pages solves this cleanly: it publishes fixed IP addresses an
apex domain can point at directly via plain A records — no nameserver control
needed — and it auto-redirects the bare apex to `www` once configured. Free, no
new account, the repo already exists.

**Consequence: DNSSEC and WHOIS privacy are never touched.** Nameservers stay at
Wix throughout. The domain registration itself can stay at Wix indefinitely — it's
a minor annual cost, not the thing driving this migration.

---

## State

| | Before | After |
|---|---|---|
| Registrar | Wix.com Ltd. | **unchanged** — still Wix |
| Nameservers | ns12/ns13.wixdns.net | **unchanged** — still Wix |
| DNSSEC / WHOIS privacy | On | **unchanged** — never touched |
| Website | Wix (185.230.63.x) | GitHub Pages |
| Email | Google Workspace | **unchanged** |

Domain expires **2026-11-20**. Nothing in this plan touches the registration, so
renew it as normal whenever that comes up — separately from all of this.

---

## 1. DNS baseline — done

See `../dns-baseline-2026-09-15.txt`. Confirmed against a full screenshot of the
Wix DNS panel: A (×3), CNAME (`www`), TXT (×2), MX (×5), NS (×2, not editable). No
DKIM, no DMARC, no SRV records exist. MX is tagged "GOOGLE WORKSPACE / MANAGED BY
WIX" — confirms Wix is the billing reseller for Workspace too (see
`../google-workspace-billing-transfer.md`).

## 2. Make the GitHub repo public — done

Required: GitHub Pages on a private repo needs a paid plan, and this org
(`Evenstar-Navigation`) is on the free plan. The repo has no secrets — it's the
public marketing site's source plus generic deploy notes — confirmed via
`git grep` before flipping it. Done 2026-09-15.

## 3. Add the CNAME file

A `CNAME` file at the repo root, containing exactly:

    www.evenstarnav.com

This is what tells GitHub Pages which domain is authoritative once Pages is
enabled — GitHub reads it and auto-configures both the custom domain and the
apex→www redirect.

## 4. Enable GitHub Pages

**Settings → Pages:**
1. Source: **Deploy from a branch**.
2. Branch: **main**, folder **/ (root)**. Save.
3. Once it builds, the custom domain field should already show
   `www.evenstarnav.com` (read from the CNAME file). If not, type it in and save.
4. Check **Enforce HTTPS** once it becomes available — it's greyed out until DNS
   (step 5) is in place and GitHub has issued a certificate, which can take up to
   ~30 minutes after the DNS records go live.

## 5. Add DNS records in the Wix panel

**Domains → Domain Actions → Manage DNS records.** You already know this panel
from step 1.

**A records** — add these four, replacing or alongside the existing apex A records
(delete the three old Wix ones — `185.230.63.171/.107/.186` — once these are in,
so the apex isn't pointing at two places):

    Host: evenstarnav.com   Value: 185.199.108.153
    Host: evenstarnav.com   Value: 185.199.109.153
    Host: evenstarnav.com   Value: 185.199.110.153
    Host: evenstarnav.com   Value: 185.199.111.153

**CNAME record** — edit the existing `www` CNAME (currently `cdn3.wixdns.net`) to:

    Host: www.evenstarnav.com   Value: evenstar-navigation.github.io

Leave MX, TXT, and NS exactly as they are — none of this touches email.

Propagation is usually minutes given the 1-hour TTLs already in place. Verify:

    dig evenstarnav.com A +short         # expect the four 185.199.10x.153 addresses
    dig www.evenstarnav.com +short       # expect evenstar-navigation.github.io
    dig evenstarnav.com MX +short        # expect the five Google entries, unchanged

## 6. Verify the live site

    curl -sI https://www.evenstarnav.com | head -1
    curl -sI https://evenstarnav.com | head -1     # should show a redirect to www

Open both in a browser. Check the About page, check images load, check the email
link. Send a test email to and from a company address as a final confirmation that
nothing on the mail side moved.

## 7. Cancel Wix hosting

Once the site's been live and correct for a few days:

- Download anything still only in the Wix media library (unlikely at this point —
  the images already used are downloaded into `img/` in this repo).
- Cancel the Wix **site/hosting plan**. This is the actual cost this whole
  migration exists to cut.
- **Do not** cancel the domain registration or touch nameservers — those stay at
  Wix. Cancelling the wrong thing here breaks DNS and email; cancelling the site
  plan is safe and is the only part that needs to go.

---

## Rollback

Entirely reversible at any point through step 6: edit the A and CNAME records back
to the values in `../dns-baseline-2026-09-15.txt`. Keep that baseline file until
this is confirmed stable.

After step 7 (Wix plan cancelled), rollback means resubscribing to Wix hosting and
reconnecting the site — more friction, but still fully possible since the domain
and DNS were never touched.

---

## Later, optional, fully separate: registrar consolidation

Nothing above requires this. If at some point the ~$10–20/year difference and
having DNS + hosting + registrar in one account is worth it, the path is:

1. Transfer the domain from Wix to any registrar that allows nameserver changes
   (Namecheap, Porkbun, etc. — not Cloudflare directly, per the constraint above).
2. Wait the ICANN-mandated 60 days.
3. Transfer from that registrar to Cloudflare (or wherever).
4. Only then does the DNSSEC/WHOIS-exposure trade-off from the original plan
   become relevant — and by then Cloudflare Registrar's free WHOIS redaction
   applies automatically once the second transfer completes.

Treat this as its own project with its own go/no-go decision, not a follow-on to
this one.

## Checklist

- [x] 1. DNS baseline recorded and confirmed against the Wix panel
- [x] 2. Repo made public
- [ ] 3. `CNAME` file committed and pushed
- [ ] 4. GitHub Pages enabled, building from `main` / root
- [x] 5. Apex A records updated to the four GitHub IPs; old Wix A records removed
- [x] 5. `www` CNAME updated to `evenstar-navigation.github.io`
- [x] 6. Site confirmed live and correct (verified via public resolvers, direct content check, and a real device on cellular data)
- [ ] 6. HTTPS cert still provisioning at GitHub — not yet enforced, needs a bit more time
- [ ] 6. Test email sent and received — confirms MX untouched
- [ ] 7. Wix hosting/site plan cancelled (registration and nameservers left alone)
