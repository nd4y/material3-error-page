# material3-error-page

A single-page nginx error page (400–511) styled in **Material 3** (baseline purple
scheme, seed `#6750A4`): a centered elevated card, an error icon badge, automatic
light/dark via `prefers-color-scheme`, and a filled "Back to home" button on 404.
No JavaScript, no external requests — the page and its variable font are fully
self-hosted.

## Files

| File | Purpose |
|---|---|
| `custom-error-page.html` | The page template. Uses nginx SSI (`ssi on;`) to fill in `$status`, `$status_text` and `$host`. |
| `custom-error-page.css` | Material 3 styling: design tokens as CSS custom properties, overridden under `@media (prefers-color-scheme: dark)`. |
| `inter-roman-var.woff2` | Self-hosted Inter variable font (weights 100–900) used for all text. |

## Requirements

- nginx built with `ngx_http_ssi_module` (present in the official `nginx` image).
- A `map` from `$status` to a human-readable `$status_text` — see below.

## Integration

```nginx
# conf.d/http-statuses.conf
map $status $status_text {
    400 'Bad Request';
    401 'Unauthorized';
    403 'Forbidden';
    404 'Not Found';
    500 'Internal Server Error';
    502 'Bad Gateway';
    503 'Service Unavailable';
    504 'Gateway Timeout';
    default 'Something went wrong';
    # ... add the status codes you actually serve
}
```

```nginx
# conf.d/error-page.conf
location ^~ /custom-error-page.html {
    ssi on;
    internal;
    root /usr/share/nginx/html/custom-error-page/;
}

location ^~ /custom-error-page.css {
    root /usr/share/nginx/html/custom-error-page/;
}

location ^~ /inter-roman-var.woff2 {
    root /usr/share/nginx/html/custom-error-page/;
}

error_page 400 401 402 403 404 405 406 407 408 409 410 411 412 413 414 415 416 417 418
           421 422 423 424 425 426 428 429 431 451 494 495 496 497
           500 501 502 503 504 505 506 507 508 510 511 /custom-error-page.html;

proxy_intercept_errors on;
```

Include both files in every `server {}` block that should use the page, and mount
this repo's three files at `/usr/share/nginx/html/custom-error-page/` (bind mount,
`COPY` in a custom image, or an init container — whatever fits your deployment).

**The `^~` matters.** These are plain-prefix locations; if your config also has a
*regex* location matching `.css`/`.woff2` (e.g. a static-asset proxy cache keyed by
extension), the regex location wins over a plain prefix by nginx's own precedence
rules and silently swallows these paths — the page's CSS/font end up proxied to
your backend instead of served from disk, and the error page renders unstyled.
`^~` forces the prefix match to win outright. (Found this the hard way running the
page behind a `location ~* \.(css|js|woff2|...)$ { proxy_cache ...; }` block.)

## Customizing

All colors are CSS custom properties on `:root` (light) and re-declared under the
dark media query — swap the hex values for your own Material 3 seed. The card
copy (404 description, generic 4xx/5xx fallback text) lives directly in
`custom-error-page.html` inside the SSI `if`/`elif` block.

## Credit

The SSI wiring pattern this page uses (`$status`/`$status_text`/`$host` template
variables, the `http-statuses.conf` status-text map) follows the approach
popularized by Johannes Siipola's MIT/CC0-licensed `nginx-custom-error-page`. The
Material 3 design, layout and CSS here are new and unrelated to that project's
styling.

## License

MIT — see [LICENSE](LICENSE) — for the HTML/CSS. `inter-roman-var.woff2` is the
[Inter](https://github.com/rsms/inter) variable font, distributed under the
SIL Open Font License 1.1.
