# ncsr-templates

Branded email and SMS templates for **NCSR** (National Cooperative Smart Registry), rendered by the
shared platform services.

## What reads these

| File | Service | Rendered by | Variables |
| --- | --- | --- | --- |
| `account-locked.ejs` | platform-auth-ms | `sendAccountLockedNotifyEmail` | `name`, `unLockInterval`, `supportEmail` *(optional)* |
| `failed-login-attempt.ejs` | platform-auth-ms | `sendFailedAttemptEmail` | `name`, `date`, `browserName` |
| `reset-password.ejs` | platform-auth-ms | `job.service` password-expiry sweep | `name`, `warningPeriodDays` |
| `otp-email.ejs` | platform-otp-ms | `otp-generation.service` | `otp`, `minutes` |
| `otp-sms.ejs` | platform-otp-ms | `otp-generation.service` | `otp`, `minutes` |

Point `TEMPLATE_PATH` at `ncsr-templates/templates` for NCSR's deployment of both services.

**`reset-password.ejs` is not a password reset.** Despite the filename, `platform-auth-ms` renders it
for the *password expiry warning* (subject key `PASSWORD_EXPIRY_NOTIFICATION`). The copy here matches
what the service actually sends. The name is kept because the service looks the file up by it.

`PWD_CHANGE_TEMPLATE_PATH` is a separate variable pointing at a **Handlebars** template used for an
SMS, not part of this folder.

## English only, and why

`platform-auth-ms` resolves the language folder against its own enum:

```ts
export enum SystemLanguage { English = 'en', French = 'fr' }
```

and falls back to `SystemLanguage.English` — that is, `en/` — whenever the requested folder is
missing. `platform-otp-ms` defaults to `environment.lang`, which is also `'en'`.

So a single `en/` folder resolves for both services whichever form of the language is passed.

**Hausa, Igbo and Yoruba are not possible here.** The platform enum has no entry for them, so those
languages would never be looked up and would silently fall back to English. NCSR's own notices —
registration, payments, membership and the rest — *are* translated into all four, because NCSR
renders those itself. This folder covers only the account-security notices the platform owns.
Widening it needs a change to `SystemLanguage` in `platform-auth-ms`. That has been raised with the
platform team.

## For DevOps — one line to confirm

Each HTML template opens with:

```js
var ASSETS = (typeof assetBaseUrl !== 'undefined' && assetBaseUrl) ? assetBaseUrl : 'https://ncsr.gov.ng/assets/email';
```

No caller passes `assetBaseUrl` today, so **the default is what renders**. It must be the public
origin serving `packages/frontend/ncsr-app/public/assets/email` from the NCSR repository — the same
five images the in-app emails use. If the portal is served from another origin, correct that one line
in each of the four HTML files.

The images are fetched by the recipient's mail client, so the origin has to be publicly reachable and
unauthenticated.

## Conventions

- **Self-contained.** Both services call `ejs.render(templateString, data)` with no `filename`, so an
  EJS include cannot be resolved. Every file carries its own copy of the shell. That is also how
  `civ-templates` and `mfa-templates` are written.
- **Tables and inline styles.** Mail clients drop `<style>` blocks and most of CSS. Images carry
  explicit `width`/`height` because Outlook otherwise sizes them at their natural width, and these
  are retina exports displayed at half size.
- **PNG, not SVG.** Gmail and Outlook both refuse SVG in a mail body.
- **`otp-sms.ejs` is plain text.** It is delivered as a text message; it must never carry markup.
- **`<%= %>`, never `<%- %>`.** The escaping form is deliberate: `name` and `browserName` arrive from
  a request, and `<%-` would render a hostile value as live markup.

## Verifying a change

```bash
node -e "
const ejs=require('ejs'), fs=require('fs');
console.log(ejs.render(fs.readFileSync('templates/en/otp-email.ejs','utf8'), { otp:'482913', minutes:10 }));
"
```

Every template in this folder was rendered with real EJS and checked for leftover tags, a missing
optional variable, and an escaped hostile `name` before being committed.
