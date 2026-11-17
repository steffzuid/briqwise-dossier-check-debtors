# Briqwise financieringscheck

Interactive lead magnet for ondernemers (debtors) on `briqwise.com/lenen/financieringscheck`. The debtor version of the broker [dossier-check](https://github.com/steffzuid/briqwise-dossier-check): same engine, rules and HubSpot properties, rewritten for an entrepreneur who borrows against their own property. Runs in Webflow, loads its code and rules from this repo through jsDelivr, and submits to HubSpot through the Forms API v3.

```
src/app.js        logic, e-mail validation, URL prefill, HubSpot connector
src/ui.html       markup + CSS (bundled into the script at build time)
rules.json        business rules, form GUIDs, e-mail settings (fetched at runtime)
dist/             built files served by jsDelivr (commit these)
test/             Playwright end-to-end test + Webflow-like fixture
```

## What differs from the broker dossier-check

| | Dossier-check (brokers) | Financieringscheck (ondernemers) |
| --- | --- | --- |
| Copy | "je klant", "case", "dossier" | "je pand", "aanvraag", "stukken" |
| Contact Type sent | `Intermediair` | `Debtor` |
| Report form | name, business e-mail, office name + postcode, type of office | name, e-mail, phone (optional), company name (optional) |
| Free-mail addresses (Gmail, Hotmail, Ziggo) | blocked | allowed. Disposable, junk and non-existent domains stay blocked |
| Intermediairsregeling | shown live and in the report | removed |
| Company object | office, with postcode for owner routing | the borrower's business, only when a name is given |
| Submit request | client consent checkbox required | no consent checkbox (they share their own data) |
| Request outside the rules (LTV > 75%, amount) | "bespreek met je accountmanager" | "Zo kan het wel" box: max. loan, extra own funds, revaluation, extra collateral (to discuss) |
| URL prefill | no | yes, see below |
| dataLayer event | `dossier_check` | `financieringscheck` |

## Webflow

Embed element where the tool should appear:

```html
<div id="bw-dc"></div>
```

Page settings → Custom code → Before `</body>`:

```html
<script src="https://cdn.jsdelivr.net/gh/steffzuid/briqwise-dossier-check-debtors@1.0.0/dist/financieringscheck.min.js"
        data-rules="https://cdn.jsdelivr.net/gh/steffzuid/briqwise-dossier-check-debtors@main/rules.json"
        data-cookieconsent="ignore" defer></script>
```

- The script is pinned to a release tag, so a code change never goes live by accident.
- `rules.json` comes from `main`, so rule changes need no Webflow publish.
- `data-cookieconsent="ignore"` stops Cookiebot auto-blocking from holding the script back. The script sets no cookies itself.
- If `rules.json` cannot be loaded within 2.5 seconds, the tool runs on the rules built into the script (the `rules.json` of that release).
- Do not place the broker dossier-check and the financieringscheck on the same page: both mount on `#bw-dc`.

## Prefill from the URL

Links from e-mails (for example the reactivation flow for rejected debtor leads) can fill in what Briqwise already knows. The tool then shows "We hebben je gegevens van eerder alvast ingevuld. Klopt het nog?".

```
/lenen/financieringscheck?waarde=800000&bedrag=500000&looptijd=120&doel=herfin&type=winkel
```

| Parameter | Value |
| --- | --- |
| `waarde` | property value in euros |
| `bedrag` | loan amount in euros (clamped to `loan.min`–`loan.max`) |
| `looptijd` | term in months, snaps to the nearest option in `loan.termOptionsMonths` |
| `doel` | `aankoop`, `herfin`, `verbouw`, `project`, `bedrijf` |
| `type` | `bedrijfsruimte`, `combi`, `woning`, `winkel`, `recreatie`, `grond` |

In a HubSpot e-mail, build the link with personalization tokens, e.g. `?waarde={{ contact.appraisal_value }}&bedrag={{ contact.desired_loan_amount }}`. Empty or invalid values are ignored. The values end up in the page URL (analytics, browser history); keep that in mind before adding more personal fields.

The `generate_lead` and `view` dataLayer events carry `dc_prefilled: 1` when the tool was prefilled, so reactivation traffic can be separated in GA4.

## Changing rules (no developer needed)

1. Edit `rules.json` on GitHub and commit to `main`.
2. Purge the cache: open `https://purge.jsdelivr.net/gh/steffzuid/briqwise-dossier-check-debtors@main/rules.json`. Without a purge, jsDelivr serves the old file for up to 12 hours.
3. Check on the live page: the root element gets `data-dc-rules="remote"` when the file loaded.

| Key | Meaning |
| --- | --- |
| `hubspot.forms.report` / `caseSubmit` | Form GUIDs. While a GUID contains `REPLACE`, nothing is sent (dry run). **Still REPLACE: run the setup script first.** |
| `hubspot.loanDurationUnit` | `months` or `years`: unit of `loan_duration_imp` on the Loan. |
| `ui.gateOutcome` | `true` hides the outcome behind the e-mail form as well. |
| `ui.meetingModalMinWidth` | From this screen width (768px) "Plan een call" opens the meeting page in the tool's own pop-up; below it, it opens the page itself. |
| `ui.meetingUrl`, `ui.phone` | "Plan een call" link and phone number in error messages. |
| `email.blockFreeMail` | `false` for ondernemers. Set to `true` to require a business address. |
| `email.mxCheck`, `email.extraBlockedDomains` | Domain check and extra blocked domains. |
| `loan.termOptionsMonths` | The terms the slider offers, in months. Keep in sync with the `dc_term_months` dropdown in HubSpot. |
| `loan.*` | Min/max amount, LTV cap (0.75), LTV above which the rate goes up (0.70), sweetspot bands. |
| `repayment.*` | Interest-only up to 50% LTV, the rest linear over 10 years; up to 24 months interest-only up to 70% LTV. |
| `costs.*`, `guarantee.*`, `rate.*` | Fees, guarantee (15% at LTV ≤ 50%), rate band and term surcharges. Same values as the broker tool. |

## HubSpot setup (once)

`scripts/hubspot-setup.mjs` reuses the dossier-check contact properties (group "Dossier-check") and creates any that are missing, then creates the forms "Financieringscheck | Rapport" and "Financieringscheck | Aanvraag indienen" with every field the tool sends. The forms have HubSpot's free-mail block list switched off. It prints the two form GUIDs for `rules.json`.

```bash
HUBSPOT_TOKEN=pat-eu1-... node scripts/hubspot-setup.mjs --dry   # shows what it would do
HUBSPOT_TOKEN=pat-eu1-... node scripts/hubspot-setup.mjs         # does it
```

Private app scopes: `crm.schemas.contacts.read`, `crm.schemas.contacts.write`, `crm.schemas.custom.read`, `forms`. Never commit the token. The script is safe to re-run.

Both tools write the same `dc_*` properties. Filter on `contact_type` (or the form) to separate brokers from ondernemers in reports and workflows.

### Workflows

Not scripted yet. The broker workflow script (postcode routing to accountmanagers, e-mails 01–03) does not apply to ondernemers. Build the debtor follow-up in HubSpot: owner assignment, the report e-mail by outcome, and the "aanvraag ontvangen" e-mail. Exclude contacts in the reactivation flow from double follow-up.

## Changing code

```bash
npm install
npm run build      # writes dist/
npm test           # needs: pip install playwright (Chromium)
```

Then bump `version` in `package.json`, rebuild, commit including `dist/`, and tag the release with the bare version number (`git tag 1.0.1`). Update the version in the Webflow script URL and publish.

Test on the live page without sending anything to HubSpot: add `?dc-dry=1` to the URL. Payloads are logged to the console.

## Notes

- The repo must be public: jsDelivr only serves public GitHub repositories. Everything in it is visible in the browser anyway; never commit private keys or tokens.
- The Google Maps key is the existing browser key from `/lenen/stap-1`. Make sure it is restricted to `*.briqwise.com` in Google Cloud.
