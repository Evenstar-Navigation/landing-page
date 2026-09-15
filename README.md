# evenstarnav.com

Static landing page for EvenStar Navigation. No build step, no dependencies.

    index.html    home
    about.html    about + team  (served at /about -- GitHub Pages resolves
                  extensionless URLs to .html natively, no config needed)
    style.css     all styles
    img/          images
    _headers      security + cache headers -- INERT on GitHub Pages (this is a
                  Cloudflare/Netlify convention). Left in place in case this ever
                  sits behind Cloudflare's proxy later; does nothing today.

## Editing

Open the HTML, change the text, commit, push. GitHub Pages redeploys on push to `main`
(usually live within a minute or two -- check the Actions tab for build status).

To preview locally:

    python3 -m http.server 8777    # then open http://localhost:8777

## Notes

- Fonts are Montserrat + Nunito Sans, loaded from Google Fonts — the same pair the
  previous Wix site used. To drop the third-party request, download the woff2 files
  into `img/` (or a `fonts/` dir) and swap the `<link>` for an `@font-face` block.
- Images came from the Wix site and were downscaled and re-encoded (8.1 MB → 1.2 MB).
  Originals are still in the Wix media library until that account is closed.
- `about.html` is reachable at `/about`, matching the old Wix URL, so existing links
  and search results keep working.

See `DEPLOY.md` for the cutover from Wix.
