# TikTok Wallet — Money Display Specification

**Owner:** Wallet Product
**Status:** Draft for engineering review
**Purpose:** Define one deterministic rule for how a user's balance is formatted (separators, decimals, digit system, currency symbol) given their country and language, including the case where that combination is not a "real" locale.

---

## 0. TL;DR (read this first)

Formatting money has two independent halves. Keep them separate and the whole problem becomes simple:

| Half | What it is | Where it comes from | Can it be wrong? |
|---|---|---|---|
| **WHAT** | the **amount** and the **currency** (`SGD 5.00`) | the **wallet ledger** | **Never.** Must be exact. |
| **HOW** | the *styling* — `1,234.56` vs `1.234,56` vs `١٬٢٣٤` and where the `$` sits | the **locale** (country + language) | Yes — degrades gracefully. |

The rule: **always pass the ledger's currency code explicitly**, and pick the locale via a **4-step fallback chain** that can never fail because it ends in `en`. If the user's language+country isn't a valid locale, the chain automatically falls back to *the country's own default locale* (so a user in Egypt still sees Egyptian formatting), and only if even that is impossible does it land on plain English.

You do **not** need a hand-maintained table of every country×language combination. The platform's `Intl` library already contains the data for ~350 locales; this spec is just the rule for *choosing which one to ask for* and *what to do when the ask doesn't exist*.

---

## 1. Background — why this is hard (plain language)

Different places write the same number differently. Three things vary:

1. **Grouping & decimal marks.** `1,234,567.89` (US/UK), `1.234.567,89` (Germany/Spain), `1 234 567,89` (France), `12,34,56,789.00` (India — grouped in *lakhs*, not thousands).
2. **Digit shapes.** Most places use Western digits `0-9`, but Egypt/Saudi Arabia use Arabic-Indic `٠١٢٣٤٥٦٧٨٩` and Iran uses Persian `۰۱۲۳۴۵۶۷۸۹`.
3. **Currency symbol & placement.** `$5`, `5 €`, `￥5`, `5 ₫`, and how many decimals a currency even has (USD = 2, JPY = 0, KWD = 3).

A **locale** is the identifier that selects all of the above. It is written `language-REGION`, e.g. `en-SG` = "English, as used in Singapore." **A locale is language + region — not just a country, and not the user's GPS location.** The same country can have several valid locales (Singapore has English, Malay, Chinese, Tamil), and the *language* drives the digit/separator style while the *region* refines it.

TikTok knows two things about a user: their **country** (region, e.g. `SG`) and their **app language** (e.g. `es`). We combine them into a locale like `es-SG`. **The edge case this spec exists to solve:** some `language + country` pairs are not real locales (there is no data for "Spanish as used in Singapore"), and some inputs are malformed. We need a rule that never crashes and never shows a nonsensical or misleading number.

---

## 2. Scope & definitions

**In scope:** formatting a single monetary balance/amount for display in the app UI.

**Out of scope:** currency conversion (FX), parsing user-typed amounts, tax/rounding business logic, and *choosing* which currency a wallet holds (that is a ledger/account decision, not a formatting decision).

| Term | Meaning |
|---|---|
| **Region** | ISO 3166-1 alpha-2 country/territory code, e.g. `SG`, `EG`, `VN`. From the user's TikTok country setting. |
| **Language** | The user's TikTok app UI language, as a BCP-47 language (optionally + script), e.g. `es`, `ar`, `zh-Hant`. |
| **Locale** | `language-REGION`, the full formatting identifier, e.g. `es-SG`. |
| **Currency** | ISO 4217 3-letter code, e.g. `SGD`, `JPY`, `KWD`. **Property of the wallet, not the country.** |
| **Minor units** | The smallest integer unit of a currency (cents). `$5.00` = `500` minor units; `¥500` = `500` minor units (JPY has 0 decimals). |
| **`Intl.NumberFormat`** | The platform-native formatter (JS/Web, and equivalent on iOS `NSNumberFormatter` / Android `android.icu.NumberFormat`). All examples use the JS API; iOS/Android have the same ICU data underneath. |

---

## 3. Inputs & outputs

**Inputs to the formatter:**

| Input | Source | Example | Required |
|---|---|---|---|
| `amountMinor` | Wallet ledger (integer) | `12345678900` | ✅ |
| `currency` | Wallet ledger (ISO 4217) | `"SGD"` | ✅ |
| `language` | TikTok app language | `"es"` | optional* |
| `region` | TikTok country setting | `"SG"` | optional* |

\* If either is missing/blank, the chain still produces a valid result (see §6, §8).

**Output:** a display string, e.g. `123.456.789,00 SGD`, plus (for logging) the locale actually used.

---

## 4. TikTok supported countries (regions)

> ⚠️ **This list is representative, not canonical.** Market availability changes and some markets are restricted or unavailable at any given time. **Engineering must source the authoritative list from TikTok's market/config service** and pass whatever `region` it provides. The formatting rule in §6 accepts **any** valid ISO 3166-1 alpha-2 code and degrades safely for unknown ones — so the exact membership of this list does not affect correctness.

Representative markets by region (ISO 3166-1 alpha-2):

- **North America:** `US` `CA` `MX`
- **Latin America:** `BR` `AR` `CL` `CO` `PE` `EC` `GT` `DO` `CR` `PA` `UY` `BO` `PY`
- **Europe:** `GB` `IE` `FR` `DE` `ES` `IT` `PT` `NL` `BE` `LU` `CH` `AT` `SE` `NO` `DK` `FI` `IS` `PL` `CZ` `SK` `HU` `RO` `BG` `GR` `HR` `SI` `RS` `UA` `LT` `LV` `EE`
- **Middle East & North Africa:** `AE` `SA` `QA` `KW` `BH` `OM` `EG` `JO` `LB` `IQ` `MA` `DZ` `TN` `IL` `TR`
- **Sub-Saharan Africa:** `NG` `KE` `GH` `ZA` `TZ` `UG` `SN` `CI` `ET`
- **South & Central Asia:** `PK` `BD` `LK` `NP` `KZ` `UZ`
- **Southeast Asia:** `SG` `MY` `ID` `TH` `VN` `PH` `KH` `MM` `LA` `BN`
- **East Asia:** `JP` `KR` `TW` `HK` `MO`
- **Oceania:** `AU` `NZ` `FJ`

*(India `IN` appears in some formatting examples in this doc purely to illustrate lakh grouping; treat market availability separately.)*

---

## 5. TikTok supported languages

> ⚠️ **Also representative.** Use the canonical app-language list from the localization team. The rule accepts any BCP-47 language subtag and degrades safely for unsupported ones.

Representative app UI languages (BCP-47 : English name):

| Code | Language | Code | Language | Code | Language |
|---|---|---|---|---|---|
| `en` | English | `pt` | Portuguese | `ta` | Tamil |
| `es` | Spanish | `ru` | Russian | `te` | Telugu |
| `fr` | French | `uk` | Ukrainian | `kn` | Kannada |
| `de` | German | `pl` | Polish | `ml` | Malayalam |
| `it` | Italian | `cs` | Czech | `ur` | Urdu |
| `nl` | Dutch | `sk` | Slovak | `fa` | Persian |
| `sv` | Swedish | `hu` | Hungarian | `he` | Hebrew |
| `da` | Danish | `ro` | Romanian | `ar` | Arabic |
| `nb` | Norwegian | `bg` | Bulgarian | `th` | Thai |
| `fi` | Finnish | `el` | Greek | `vi` | Vietnamese |
| `tr` | Turkish | `hr` | Croatian | `km` | Khmer |
| `id` | Indonesian | `sr` | Serbian | `my` | Burmese |
| `ms` | Malay | `hi` | Hindi | `ja` | Japanese |
| `fil` | Filipino | `bn` | Bengali | `ko` | Korean |
| `zh-Hant` | Chinese (Traditional) | `gu` | Gujarati | `mr` | Marathi |
| `pa` | Punjabi | | | | |

**Note on Chinese:** always carry the **script** subtag (`zh-Hant` for Traditional, `zh-Hans` for Simplified), not a region. Script — not region — is what selects the right character set.

---

## 6. The rule

### 6.1 Principle: separate WHAT from HOW

- **WHAT** = `amountMinor` + `currency`, both from the ledger. Passed explicitly. Always exact.
- **HOW** = the locale, chosen by the fallback chain below. Only affects presentation.

Never infer the currency from the country. A user physically in Singapore may hold a `USD` balance; the currency is whatever the **ledger** says.

### 6.2 Step 1 — Normalize the inputs

- `region`: trim, uppercase. (`sg` → `SG`)
- `language`: trim, lowercase, replace `_` with `-`. **Keep a script subtag** (4 letters, e.g. `Hant`) but **drop any region** the language field may carry (a 2-letter or 3-digit subtag). This prevents a legacy value like `zh_CN` from silently overriding the real region.
  - `zh_Hant` → `zh-Hant` (keep script)
  - `zh_CN` → `zh` (drop the stray region `CN`; the real region comes from `region`)

### 6.3 Step 2 — Build the locale fallback chain

Produce an **ordered candidate list**, dropping any tag that is not well-formed, then let `Intl` pick the first one it has data for:

| # | Candidate | Why | Example (lang=`es`, region=`SG`) |
|---|---|---|---|
| 1 | `language-REGION` | Exact — the user's language as used in their country | `es-SG` |
| 2 | `language` | Keep the **user's reading conventions** even if #1 has no data | `es` |
| 3 | **country default** = `defaultLanguageOf(REGION)-REGION` | Language unsupported → use the **country's own** convention | `en-SG` |
| 4 | `en` | Guaranteed anchor — the call can never fail | `en` |

The "country default" in #3 is computed automatically from the region using CLDR's *likely-subtags* data (`Intl.Locale("und-"+REGION).maximize()`), e.g. `SG→en`, `EG→ar`, `VN→vi`, `IN→hi`. **No hand-maintained table needed.**

> **Why #3 matters (the core edge case):** if you asked `Intl` for an unsupported locale like `xyz-SG` directly, it would throw away the `-SG` and jump to *the device's default locale* (often `en-US`) — ignoring Singapore entirely. Step 3 intercepts that and substitutes the **country's** default locale, so formatting stays locally appropriate.

### 6.4 Step 3 — Format with the currency

Create the formatter with the chain and the ledger currency; `Intl` applies the correct fraction digits automatically (do **not** hardcode `2`):

```
new Intl.NumberFormat(chain, { style: "currency", currency })
```

Convert the integer minor units to a **decimal string** (never a float) using the currency's exponent, then format that string.

### 6.5 Reference implementation (JavaScript)

```js
// --- Step 1: normalize ---
function normalizeLanguage(raw) {
  const parts = (raw || "").trim().replace(/_/g, "-").split("-").filter(Boolean);
  if (!parts.length) return "";
  const lang = parts[0].toLowerCase();
  const script = parts.find(p => /^[A-Za-z]{4}$/.test(p)); // keep script, drop region
  return script ? `${lang}-${script[0].toUpperCase()}${script.slice(1).toLowerCase()}` : lang;
}

// --- Step 2: fallback chain ---
function resolveWalletLocales(language, region) {
  const lang = normalizeLanguage(language);
  const reg  = (region || "").trim().toUpperCase();
  const out = [];
  const push = t => { try { new Intl.Locale(t); out.push(t); } catch {} }; // drop malformed
  if (lang && reg) push(`${lang}-${reg}`);                     // 1
  if (lang)        push(lang);                                 // 2
  if (reg) { try {                                             // 3 country default
    const m = new Intl.Locale("und-" + reg).maximize();
    push(`${m.language}-${reg}`);   // language-REGION (no script — script can block digit data)
  } catch {} }
  push("en");                                                  // 4 anchor
  return out;
}

// --- Step 3: money → string, no float ---
function minorToDecimalString(minor, exponent) {
  const neg = minor < 0n;
  let s = (neg ? -minor : minor).toString();
  if (exponent === 0) return (neg ? "-" : "") + s;
  while (s.length <= exponent) s = "0" + s;
  return (neg ? "-" : "") + s.slice(0, -exponent) + "." + s.slice(-exponent);
}

function formatWalletMoney(amountMinor, currency, language, region, { forceLatinDigits = false } = {}) {
  const locales = resolveWalletLocales(language, region);
  const opts = { style: "currency", currency };
  if (forceLatinDigits) opts.numberingSystem = "latn";
  const nf = new Intl.NumberFormat(locales, opts);
  const exponent = nf.resolvedOptions().maximumFractionDigits;         // from ISO 4217
  const text = nf.format(minorToDecimalString(BigInt(amountMinor), exponent));
  return { text, localeUsed: nf.resolvedOptions().locale };
}
```

iOS/Android use the same ICU data; port the three steps to `NSNumberFormatter` / `android.icu.number.NumberFormatter` with the same candidate list and currency handling.

---

## 7. Worked examples

All outputs below are **real `Intl` output** for amount `123456789.00` in each wallet's currency (VND/KWD rows use their own amounts to show 0- and 3-decimal handling).

| Language | Region | Currency | Output | Locale used | Note |
|---|---|---|---|---|---|
| `en` | `US` | USD | `$123,456,789.00` | `en-US` | baseline |
| `en` | `GB` | GBP | `£123,456,789.00` | `en-GB` | |
| `de` | `DE` | EUR | `123.456.789,00 €` | `de-DE` | dot groups, comma decimal, symbol after |
| `fr` | `FR` | EUR | `123 456 789,00 €` | `fr-FR` | space grouping |
| `es` | `SG` | SGD | `123.456.789,00 SGD` | `es` | **#1 `es-SG` has no data → #2 `es`**: Spanish style kept |
| `en` | `SG` | SGD | `$123,456,789.00` | `en-SG` | |
| `zh-Hant` | `TW` | TWD | `$123,456,789.00` | `zh-Hant-TW` | script selects Traditional |
| `ja` | `JP` | JPY | `￥12,345,678,900` | `ja-JP` | **0 decimals** |
| `hi` | `IN` | INR | `₹12,34,56,789.00` | `hi-IN` | **lakh grouping** |
| `vi` | `VN` | VND | `12.345.678.900 ₫` | `vi-VN` | 0 decimals |
| `th` | `TH` | THB | `฿123,456,789.00` | `th-TH` | |
| `ar` | `KW` | KWD | `١٢٬٣٤٥٬٦٧٨٫٩٠٠ د.ك.‏` | `ar-KW` | **Arabic-Indic digits, 3 decimals, RTL** |
| `ar` | `EG` | EGP | `١٢٣٬٤٥٦٬٧٨٩٫٠٠ ج.م.‏` | `ar-EG` | native digits |
| `fa` | `IR` | IRR | `ریال ۱۲٬۳۴۵٬۶۷۸٬۹۰۰` | `fa-IR` | **Persian digits** |

---

## 8. Edge cases & how to handle them

| # | Situation | Input example | Chain | Result | Rule |
|---|---|---|---|---|---|
| E1 | **Language not valid for that country** (the main case) | lang=`xyz`, reg=`SG` | `["xyz-SG","xyz","en-SG","en"]` | `$123,456,789.00` via **`en-SG`** | #1/#2 have no data → **#3 country default** = `en-SG`. Country formatting preserved. |
| E2 | Unsupported language, **non-Latin-digit country** | lang=`xyz`, reg=`EG` | `[…,"ar-EG","en"]` | `١٢٣٬٤٥٦٬٧٨٩٫٠٠ ج.م.‏` via **`ar-EG`** | Country default correctly yields **native Egyptian digits**. |
| E3 | Unsupported language, **0-decimal currency** | lang=`xyz`, reg=`VN`, cur=`VND` | `[…,"vi-VN","en"]` | `12.345.678.900 ₫` | Country default `vi-VN`; VND fraction digits (0) from ISO 4217. |
| E4 | **Both** language and region unknown | lang=`xyz`, reg=`ZZ` | `["xyz-ZZ","xyz","en-ZZ","en"]` | `$123,456,789.00` via **`en`** | Nothing matches → anchor `en`. Never crashes. |
| E5 | **Missing** language and region (nulls) | lang=`∅`, reg=`∅` | `["en"]` | `$123,456,789.00` via **`en`** | Chain still valid. |
| E6 | **Malformed / legacy** language code | lang=`zh_CN`, reg=`SG` | normalize → `zh`; `["zh-SG","zh","en-SG","en"]` | uses `zh-SG` | Normalization **strips the stray region** `CN` so it can't override `SG`. Without this fix it would wrongly format as `zh-CN`. |
| E7 | Language needs a **script** | lang=`zh-Hant`, reg=`HK` | `["zh-hant-HK","zh-hant","zh-HK","en"]` | Traditional Chinese | Keep the script subtag; it selects the character set. |
| E8 | **Currency ≠ country's usual currency** | reg=`SG`, cur=`USD` | (locale from SG) | `$123,456,789.00` in **USD** | Currency comes from the **ledger**, never inferred from region. |
| E9 | **Invalid currency code** | cur=`"XYZ"` | — | `Intl` throws `RangeError` | Validate `currency` against ISO 4217 **before** formatting; treat as a data error, show a safe placeholder, alert monitoring. **Do not** silently swap currencies. |
| E10 | Truly malformed language tag (bad characters) | lang=`"e!"` | dropped by `push()` guard | falls through to country default / `en` | The try/catch guard drops un-parseable tags instead of throwing. |

**Golden behavior:** in every row above, the **amount and currency are exact**. Only the *styling* falls back. That is the property to preserve.

---

## 9. Product decisions to lock (policy, not code)

These are choices only Product can make. Decide once, document, apply everywhere.

1. **Digit system — native vs forced Latin.**
   By default the locale decides: `ar-EG` → `١٢٣`, `fa-IR` → `۱۲۳`, everyone else → `123`.
   - *Option A (recommended):* respect native digits — it's what the user reads.
   - *Option B:* force Western digits everywhere (`forceLatinDigits: true` → `numberingSystem:"latn"`) for consistency, fraud-review readability, or CS tooling.
   **Decision required. Recommendation: A, with B available as a per-surface flag.**

2. **Amount precision.** Store money as **integer minor units**; format via a **decimal string** (see §6.5). Never `amount/100` in floating point — it corrupts large balances. Use 64-bit/`BigInt` end to end.

3. **Fraction digits** come from **ISO 4217 via `Intl`** (USD 2, JPY 0, KWD 3). Never hardcode 2. If Product wants to always show 2 decimals for UI consistency, that is an *override* (`minimumFractionDigits`/`maximumFractionDigits`) and must be an explicit, documented decision — be aware it makes JPY look like `¥500.00`.

4. **Bidi / RTL safety.** Arabic/Hebrew/Persian output is right-to-left and contains directional marks. When embedding a money string inside otherwise-LTR UI (or vice versa), wrap it in a Unicode isolate (`⁨…⁩`) or a `dir`-scoped element so the sign/symbol doesn't visually jump. Required for MENA markets.

5. **Rounding for display.** This spec **displays** the ledger value; it does not round for business purposes. If a balance has more precision than the currency's minor unit, define separately whether to truncate or round — do not let the formatter silently decide.

---

## 10. Test vectors (for QA — golden outputs)

Amount = `12345678900` minor units unless noted. These are the exact strings the reference implementation produces; use them as regression fixtures.

```
lang    region currency  expected                       locale_used
en      US     USD       $123,456,789.00                en-US
de      DE     EUR       123.456.789,00 €               de-DE
fr      FR     EUR       123 456 789,00 €               fr-FR
es      SG     SGD       123.456.789,00 SGD             es
en      SG     SGD       $123,456,789.00                en-SG
ja      JP     JPY       ￥12,345,678,900                ja-JP
hi      IN     INR       ₹12,34,56,789.00               hi-IN
vi      VN     VND       12.345.678.900 ₫               vi-VN
ar      EG     EGP       ١٢٣٬٤٥٦٬٧٨٩٫٠٠ ج.م.‏            ar-EG
ar      KW     KWD       ١٢٬٣٤٥٬٦٧٨٫٩٠٠ د.ك.‏            ar-KW   (3 decimals)
fa      IR     IRR       ریال ۱۲٬۳۴۵٬۶۷۸٬۹۰۰            fa-IR
xyz     SG     SGD       $123,456,789.00                en-SG   (E1 fallback)
xyz     EG     EGP       ١٢٣٬٤٥٦٬٧٨٩٫٠٠ ج.م.‏            ar-EG   (E2 fallback)
xyz     ZZ     USD       $123,456,789.00                en      (E4 anchor)
∅       ∅      USD       $123,456,789.00                en      (E5 anchor)
```

Extra fixtures: `VND` minor `1234568` → `1.234.568 ₫` (0 decimals); `KWD` minor `1234567890` → `١٬٢٣٤٬٥٦٧٫٨٩٠ د.ك.‏` (3 decimals).

---

## 11. Non-goals

- Not a currency-conversion spec (no FX rates).
- Not a canonical source for TikTok's market or language lists — those come from platform config (§4, §5 are illustrative).
- Does not choose which currency a wallet holds — that is a ledger decision.

---

## Appendix — FAQ

**Q: Why not just build a big table of country×language → format?**
Because the platform (`Intl`/ICU) already ships that data for ~350 locales and updates it. Hand-maintaining it would drift and rot. This spec only defines *which locale to request* and *the fallback when the request has no data*.

**Q: What does `Intl.NumberFormat().resolvedOptions().locale` return, and should we use it?**
It returns the **device/runtime default locale** (from the OS/browser language settings, *not* the user's GPS country). **Do not** rely on it for wallet formatting — always pass the explicit chain from §6, so the output depends on the user's TikTok settings, not their handset's OS language.

**Q: A Spanish user in Singapore — what do they see?**
`es-SG` has no dedicated data, so the chain falls to `es`: `123.456.789,00 SGD` (Spanish separators, SGD currency). The **currency stays SGD**; only the styling is Spanish. Correct and safe.

**Q: An unsupported language in Egypt?**
Chain falls to the **country default** `ar-EG` → native Arabic-Indic digits and the EGP symbol. The user still gets locally-appropriate formatting despite the unknown language.

**Q: Can this ever throw or show a blank?**
Only if the **currency code itself is invalid** (E9) — validate it upstream. The locale chain always terminates in `en`, so locale resolution never fails.
