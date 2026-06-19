# v0.2.1 — Bug fixes: buttons, layout, Jinja2 compatibility

## Bug fixes

- **Buttons now work** — all `hass.callService` calls now use `document.querySelector('home-assistant').hass` instead of a bare `hass` reference, which is not in scope inside `html-template-card`'s shadow DOM
- **Blank card fixed** — `(function(){{% if` inside onclick attributes created the sequence `{{%` which Jinja2's subscription engine lexed as an unclosed variable expression followed by unexpected `%`; fixed by rewriting conditionals as JS-level guards using `{{ 'true' if condition else 'false' }}`
- **CSS compatibility** — `border-radius:50%` changed to `border-radius:50px`; `now().strftime('%H:%M')` replaced with manual hour/minute string construction; `%7` modulo replaced with conditional arithmetic — all to avoid bare `%` characters in positions that could interact with Jinja2 delimiters

## Layout change

- **Winterise ❄ and Disarm ■ buttons moved to the header** alongside the rain toggle and badge — these are infrequent seasonal/override controls and make more sense at the top rather than competing with Go
- **Go button is now full-width** in its own row — it is the primary daily action and gets the visual prominence to match

## Upgrade

Replace your existing card YAML with `releases/v0.2.1/card.yaml`. No new helpers required.
