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

## Appendix A — FAQ

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

---

## Appendix B — All locales → number format (defaults highlighted)

Every CLDR locale the formatter can resolve to, with the plain-number sample `1234567.89`. Generated live from `Intl` — regenerate rather than hand-edit.

**Legend:** ★ = this locale is a **language's default** (what fallback step 2 lands on). ◆ = this locale is a **region's default** (what fallback step 3 lands on). Rows carrying either marker are shown in **bold**.

### Africa

| Locale | Language | Sample `1234567.89` | Digits | Default of |
|---|---|---|---|---|
| `af` | Afrikaans | 1 234 567,89 | latn |  |
| `af-NA` | Afrikaans (Namibia) | 1 234 567,89 | latn |  |
| `am` | Amharic | 1,234,567.89 | latn |  |
| **`ar-DZ`** | Arabic (Algeria) | **1.234.567,89** | latn | ◆ region `DZ` |
| **`ar-EG`** | Arabic (Egypt) | **١٬٢٣٤٬٥٦٧٫٨٩** | arab | ★ lang `ar` · ◆ region `EG` |
| `ar-LY` | Arabic (Libya) | 1.234.567,89 | latn |  |
| **`ar-MA`** | Arabic (Morocco) | **1.234.567,89** | latn | ◆ region `MA` |
| `ar-SD` | Arabic (Sudan) | ١٬٢٣٤٬٥٦٧٫٨٩ | arab |  |
| `ar-SS` | Arabic (South Sudan) | ١٬٢٣٤٬٥٦٧٫٨٩ | arab |  |
| **`ar-TN`** | Arabic (Tunisia) | **1.234.567,89** | latn | ◆ region `TN` |
| `ber` |  | 1,234,567.89 | latn |  |
| `bm` | Bambara | 1,234,567.89 | latn |  |
| `dyo` | Jola-Fonyi | 1 234 567,89 | latn |  |
| `ee` | Ewe | 1,234,567.89 | latn |  |
| `ff` | Fula | 1 234 567,89 | latn |  |
| `ff-Latn` | Fula (Latin) | 1 234 567,89 | latn |  |
| `ha` | Hausa | 1,234,567.89 | latn |  |
| `ha-GH` | Hausa (Ghana) | 1,234,567.89 | latn |  |
| `ha-NE` | Hausa (Niger) | 1,234,567.89 | latn |  |
| `ig` | Igbo | 1,234,567.89 | latn |  |
| `kab` | Kabyle | 1 234 567,89 | latn |  |
| `khq` | Koyra Chiini | 1 234 567.89 | latn |  |
| `ki` | Kikuyu | 1,234,567.89 | latn |  |
| `kln` | Kalenjin | 1,234,567.89 | latn |  |
| `ksf` | Bafia | 1 234 567,89 | latn |  |
| `lg` | Ganda | 1,234,567.89 | latn |  |
| `ln` | Lingala | 1.234.567,89 | latn |  |
| `lu` | Luba-Katanga | 1.234.567,89 | latn |  |
| `luo` | Luo | 1,234,567.89 | latn |  |
| `mg` | Malagasy | 1,234,567.89 | latn |  |
| `mfe` | Morisyen | 1 234 567.89 | latn |  |
| `nd` | North Ndebele | 1,234,567.89 | latn |  |
| `nus` | Nuer | 1,234,567.89 | latn |  |
| `nyn` | Nyankole | 1,234,567.89 | latn |  |
| `om` | Oromo | 1,234,567.89 | latn |  |
| `om-KE` | Oromo (Kenya) | 1,234,567.89 | latn |  |
| `rn` | Rundi | 1.234.567,89 | latn |  |
| `rw` | Kinyarwanda | 1.234.567,89 | latn |  |
| `saq` | Samburu | 1,234,567.89 | latn |  |
| `sbp` | Sangu | 1,234,567.89 | latn |  |
| `seh` | Sena | 1.234.567,89 | latn |  |
| `sg` | Sango | 1.234.567,89 | latn |  |
| `sn` | Shona | 1,234,567.89 | latn |  |
| `so` | Somali | 1,234,567.89 | latn |  |
| `so-DJ` | Somali (Djibouti) | 1,234,567.89 | latn |  |
| `so-ET` | Somali (Ethiopia) | 1,234,567.89 | latn |  |
| `so-KE` | Somali (Kenya) | 1,234,567.89 | latn |  |
| `sw` | Swahili | 1,234,567.89 | latn |  |
| `sw-CD` | Congo Swahili | 1.234.567,89 | latn |  |
| **`sw-KE`** | Swahili (Kenya) | **1,234,567.89** | latn | ◆ region `KE` |
| **`sw-UG`** | Swahili (Uganda) | **1,234,567.89** | latn | ◆ region `UG` |
| `teo` | Teso | 1,234,567.89 | latn |  |
| `teo-KE` | Teso (Kenya) | 1,234,567.89 | latn |  |
| `ti` | Tigrinya | 1,234,567.89 | latn |  |
| `ti-ER` | Tigrinya (Eritrea) | 1,234,567.89 | latn |  |
| `twq` | Tasawaq | 1 234 567.89 | latn |  |
| `vai` | Vai | 1,234,567.89 | latn |  |
| `vai-Latn` | Vai (Latin) | 1,234,567.89 | latn |  |
| `wo` | Wolof | 1.234.567,89 | latn |  |
| `xh` | Xhosa | 1 234 567.89 | latn |  |
| `yav` | Yangben | 1 234 567,89 | latn |  |
| `yo` | Yoruba | 1,234,567.89 | latn |  |
| `yo-BJ` | Yoruba (Benin) | 1,234,567.89 | latn |  |
| `zgh` | Standard Moroccan Tamazight | 1 234 567,89 | latn |  |
| `zu` | Zulu | 1,234,567.89 | latn |  |

### Americas

| Locale | Language | Sample `1234567.89` | Digits | Default of |
|---|---|---|---|---|
| `arn` | Mapuche | 1,234,567.89 | latn |  |
| `ay` | Aymara | 1,234,567.89 | latn |  |
| `chr` | Cherokee | 1,234,567.89 | latn |  |
| **`en-CA`** | Canadian English | **1,234,567.89** | latn | ◆ region `CA` |
| **`en-US`** | American English | **1,234,567.89** | latn | ★ lang `en` · ◆ region `US` |
| `es-419` | Latin American Spanish | 1,234,567.89 | latn |  |
| **`es-AR`** | Spanish (Argentina) | **1.234.567,89** | latn | ◆ region `AR` |
| **`es-BO`** | Spanish (Bolivia) | **1.234.567,89** | latn | ◆ region `BO` |
| `es-BR` | Spanish (Brazil) | 1,234,567.89 | latn |  |
| `es-BZ` | Spanish (Belize) | 1,234,567.89 | latn |  |
| **`es-CL`** | Spanish (Chile) | **1.234.567,89** | latn | ◆ region `CL` |
| **`es-CO`** | Spanish (Colombia) | **1.234.567,89** | latn | ◆ region `CO` |
| **`es-CR`** | Spanish (Costa Rica) | **1 234 567,89** | latn | ◆ region `CR` |
| `es-CU` | Spanish (Cuba) | 1,234,567.89 | latn |  |
| **`es-DO`** | Spanish (Dominican Republic) | **1,234,567.89** | latn | ◆ region `DO` |
| **`es-EC`** | Spanish (Ecuador) | **1.234.567,89** | latn | ◆ region `EC` |
| **`es-GT`** | Spanish (Guatemala) | **1,234,567.89** | latn | ◆ region `GT` |
| `es-HN` | Spanish (Honduras) | 1,234,567.89 | latn |  |
| **`es-MX`** | Mexican Spanish | **1,234,567.89** | latn | ◆ region `MX` |
| `es-NI` | Spanish (Nicaragua) | 1,234,567.89 | latn |  |
| **`es-PA`** | Spanish (Panama) | **1,234,567.89** | latn | ◆ region `PA` |
| **`es-PE`** | Spanish (Peru) | **1,234,567.89** | latn | ◆ region `PE` |
| `es-PR` | Spanish (Puerto Rico) | 1,234,567.89 | latn |  |
| `es-PY` | Spanish (Paraguay) | 1.234.567,89 | latn |  |
| `es-SV` | Spanish (El Salvador) | 1,234,567.89 | latn |  |
| `es-US` | Spanish (United States) | 1,234,567.89 | latn |  |
| **`es-UY`** | Spanish (Uruguay) | **1.234.567,89** | latn | ◆ region `UY` |
| `es-VE` | Spanish (Venezuela) | 1.234.567,89 | latn |  |
| `fr-CA` | Canadian French | 1 234 567,89 | latn |  |
| `gn` | Guarani | 1,234,567.89 | latn |  |
| `ht` | Haitian Creole | 1,234,567.89 | latn |  |
| `kgp` | Kaingang | 1.234.567,89 | latn |  |
| `lkt` | Lakota | 1,234,567.89 | latn |  |
| `moh` | Mohawk | 1,234,567.89 | latn |  |
| `nv` | Navajo | 1,234,567.89 | latn |  |
| **`pt-BR`** | Brazilian Portuguese | **1.234.567,89** | latn | ★ lang `pt` · ◆ region `BR` |
| `qu` | Quechua | 1,234,567.89 | latn |  |
| `qu-BO` | Quechua (Bolivia) | 1.234.567,89 | latn |  |
| `qu-EC` | Quechua (Ecuador) | 1,234,567.89 | latn |  |
| `yrl` | Nheengatu | 1.234.567,89 | latn |  |

### Asia

| Locale | Language | Sample `1234567.89` | Digits | Default of |
|---|---|---|---|---|
| `en-HK` | English (Hong Kong SAR China) | 1,234,567.89 | latn |  |
| `en-IN` | English (India) | 12,34,567.89 | latn |  |
| `en-MY` | English (Malaysia) | 1,234,567.89 | latn |  |
| `en-PH` | English (Philippines) | 1,234,567.89 | latn |  |
| `en-PK` | English (Pakistan) | 1,234,567.89 | latn |  |
| **`en-SG`** | English (Singapore) | **1,234,567.89** | latn | ◆ region `SG` |
| `as` | Assamese | ১২,৩৪,৫৬৭.৮৯ | beng |  |
| **`bn`** | Bangla | **১২,৩৪,৫৬৭.৮৯** | beng | ★ lang `bn` |
| `bn-IN` | Bangla (India) | ১২,৩৪,৫৬৭.৮৯ | beng |  |
| `bo` | Tibetan | 1,234,567.89 | latn |  |
| `bo-IN` | Tibetan (India) | 1,234,567.89 | latn |  |
| `brx` | Bodo | 12,34,567.89 | latn |  |
| `ccp` | Chakma | 𑄷𑄸,𑄹𑄺,𑄻𑄼𑄽.𑄾𑄿 | cakm |  |
| `dz` | Dzongkha | ༡༢,༣༤,༥༦༧.༨༩ | tibt |  |
| **`gu`** | Gujarati | **12,34,567.89** | latn | ★ lang `gu` |
| **`hi`** | Hindi | **12,34,567.89** | latn | ★ lang `hi` |
| `hi-Latn` | Hindi (Latin) | 12,34,567.89 | latn |  |
| `hy` | Armenian | 1 234 567,89 | latn |  |
| **`id`** | Indonesian | **1.234.567,89** | latn | ★ lang `id` |
| `jv` | Javanese | 1.234.567,89 | latn |  |
| `ka` | Georgian | 1 234 567,89 | latn |  |
| `kk` | Kazakh | 1 234 567,89 | latn |  |
| **`km`** | Khmer | **1,234,567.89** | latn | ★ lang `km` |
| **`kn`** | Kannada | **1,234,567.89** | latn | ★ lang `kn` |
| **`ko`** | Korean | **1,234,567.89** | latn | ★ lang `ko` |
| `ko-KP` | Korean (North Korea) | 1,234,567.89 | latn |  |
| `ks` | Kashmiri | ۱٬۲۳۴٬۵۶۷٫۸۹ | arabext |  |
| `ks-Deva` | Kashmiri (Devanagari) | 1,234,567.89 | latn |  |
| `ku` | Kurdish | 1.234.567,89 | latn |  |
| `ky` | Kyrgyz | 1 234 567,89 | latn |  |
| `lo` | Lao | 1.234.567,89 | latn |  |
| `mai` | Maithili | 1,234,567.89 | latn |  |
| **`ml`** | Malayalam | **12,34,567.89** | latn | ★ lang `ml` |
| `mn` | Mongolian | 1,234,567.89 | latn |  |
| `mni` | Manipuri | ১,২৩৪,৫৬৭.৮৯ | beng |  |
| **`mr`** | Marathi | **१२,३४,५६७.८९** | deva | ★ lang `mr` |
| **`ms`** | Malay | **1,234,567.89** | latn | ★ lang `ms` |
| **`ms-BN`** | Malay (Brunei) | **1.234.567,89** | latn | ◆ region `BN` |
| `ms-ID` | Malay (Indonesia) | 1.234.567,89 | latn |  |
| `ms-SG` | Malay (Singapore) | 1,234,567.89 | latn |  |
| **`my`** | Burmese | **၁,၂၃၄,၅၆၇.၈၉** | mymr | ★ lang `my` |
| `ne` | Nepali | १२,३४,५६७.८९ | deva |  |
| `ne-IN` | Nepali (India) | १२,३४,५६७.८९ | deva |  |
| `or` | Odia | 12,34,567.89 | latn |  |
| **`pa`** | Punjabi | **12,34,567.89** | latn | ★ lang `pa` |
| `pa-Arab` | Punjabi (Arabic) | ۱٬۲۳۴٬۵۶۷٫۸۹ | arabext |  |
| `ps` | Pashto | ۱٬۲۳۴٬۵۶۷٫۸۹ | arabext |  |
| `ps-PK` | Pashto (Pakistan) | ۱٬۲۳۴٬۵۶۷٫۸۹ | arabext |  |
| `sa` | Sanskrit | १२,३४,५६७.८९ | deva |  |
| `sat` | Santali | ᱑,᱒᱓᱔,᱕᱖᱗.᱘᱙ | olck |  |
| `sd` | Sindhi | ١٬٢٣٤٬٥٦٧.٨٩ | arab |  |
| `sd-Deva` | Sindhi (Devanagari) | 1,234,567.89 | latn |  |
| `si` | Sinhala | 1,234,567.89 | latn |  |
| `su` | Sundanese | 1.234.567,89 | latn |  |
| **`ta`** | Tamil | **12,34,567.89** | latn | ★ lang `ta` |
| `ta-LK` | Tamil (Sri Lanka) | 12,34,567.89 | latn |  |
| `ta-MY` | Tamil (Malaysia) | 1,234,567.89 | latn |  |
| `ta-SG` | Tamil (Singapore) | 1,234,567.89 | latn |  |
| **`te`** | Telugu | **12,34,567.89** | latn | ★ lang `te` |
| `tg` | Tajik | 1 234 567,89 | latn |  |
| **`th`** | Thai | **1,234,567.89** | latn | ★ lang `th` |
| `tk` | Turkmen | 1 234 567,89 | latn |  |
| `tt` | Tatar | 1 234 567,89 | latn |  |
| `ug` | Uyghur | 1,234,567.89 | latn |  |
| **`ur`** | Urdu | **1,234,567.89** | latn | ★ lang `ur` |
| `ur-IN` | Urdu (India) | ۱٬۲۳۴٬۵۶۷٫۸۹ | arabext |  |
| `uz` | Uzbek | 1 234 567,89 | latn |  |
| `uz-Arab` | Uzbek (Arabic) | ۱٬۲۳۴٬۵۶۷٫۸۹ | arabext |  |
| `uz-Cyrl` | Uzbek (Cyrillic) | 1 234 567,89 | latn |  |
| **`vi`** | Vietnamese | **1.234.567,89** | latn | ★ lang `vi` |
| `yue` | Cantonese | 1,234,567.89 | latn |  |
| `yue-Hans` | Cantonese (Simplified) | 1,234,567.89 | latn |  |
| **`zh`** | Chinese | **1,234,567.89** | latn | ★ lang `zh-Hans` |
| **`zh-Hans`** | Simplified Chinese | **1,234,567.89** | latn | ★ lang `zh-Hans` |
| `zh-Hans-HK` | Chinese (Simplified, Hong Kong SAR China) | 1,234,567.89 | latn |  |
| `zh-Hans-MO` | Chinese (Simplified, Macao SAR China) | 1,234,567.89 | latn |  |
| `zh-Hans-SG` | Chinese (Simplified, Singapore) | 1,234,567.89 | latn |  |
| **`zh-Hant`** | Traditional Chinese | **1,234,567.89** | latn | ★ lang `zh-Hant` |
| **`zh-Hant-HK`** | Chinese (Traditional, Hong Kong SAR China) | **1,234,567.89** | latn | ◆ region `HK` |
| **`zh-Hant-MO`** | Chinese (Traditional, Macao SAR China) | **1,234,567.89** | latn | ◆ region `MO` |
| **`ja`** | Japanese | **1,234,567.89** | latn | ★ lang `ja` |

### Europe

| Locale | Language | Sample `1234567.89` | Digits | Default of |
|---|---|---|---|---|
| `be` | Belarusian | 1 234 567,89 | latn |  |
| `be-tarask` | Belarusian (Taraskievica orthography) | 1 234 567,89 | latn |  |
| **`bg`** | Bulgarian | **1 234 567,89** | latn | ★ lang `bg` |
| `br` | Breton | 1 234 567,89 | latn |  |
| `bs` | Bosnian | 1.234.567,89 | latn |  |
| `bs-Cyrl` | Bosnian (Cyrillic) | 1.234.567,89 | latn |  |
| `ca` | Catalan | 1.234.567,89 | latn |  |
| `ca-AD` | Catalan (Andorra) | 1.234.567,89 | latn |  |
| `ca-FR` | Catalan (France) | 1.234.567,89 | latn |  |
| `ca-IT` | Catalan (Italy) | 1.234.567,89 | latn |  |
| **`cs`** | Czech | **1 234 567,89** | latn | ★ lang `cs` |
| `cv` | Chuvash | 1 234 567,89 | latn |  |
| `cy` | Welsh | 1,234,567.89 | latn |  |
| **`da`** | Danish | **1.234.567,89** | latn | ★ lang `da` |
| `da-GL` | Danish (Greenland) | 1.234.567,89 | latn |  |
| **`de`** | German | **1.234.567,89** | latn | ★ lang `de` |
| **`de-AT`** | Austrian German | **1 234 567,89** | latn | ◆ region `AT` |
| `de-BE` | German (Belgium) | 1.234.567,89 | latn |  |
| **`de-CH`** | Swiss High German | **1'234'567.89** | latn | ◆ region `CH` |
| `de-IT` | German (Italy) | 1.234.567,89 | latn |  |
| `de-LI` | German (Liechtenstein) | 1'234'567.89 | latn |  |
| `de-LU` | German (Luxembourg) | 1.234.567,89 | latn |  |
| `dsb` | Lower Sorbian | 1.234.567,89 | latn |  |
| **`el`** | Greek | **1.234.567,89** | latn | ★ lang `el` |
| `el-CY` | Greek (Cyprus) | 1.234.567,89 | latn |  |
| **`en-GB`** | British English | **1,234,567.89** | latn | ◆ region `GB` |
| **`en-IE`** | English (Ireland) | **1,234,567.89** | latn | ◆ region `IE` |
| `en-MT` | English (Malta) | 1,234,567.89 | latn |  |
| **`es-ES`** | European Spanish | **1.234.567,89** | latn | ★ lang `es` · ◆ region `ES` |
| `et` | Estonian | 1 234 567,89 | latn |  |
| `eu` | Basque | 1.234.567,89 | latn |  |
| **`fi`** | Finnish | **1 234 567,89** | latn | ★ lang `fi` |
| `fo` | Faroese | 1.234.567,89 | latn |  |
| `fo-DK` | Faroese (Denmark) | 1.234.567,89 | latn |  |
| **`fr`** | French | **1 234 567,89** | latn | ★ lang `fr` |
| `fr-BE` | French (Belgium) | 1 234 567,89 | latn |  |
| `fr-CH` | Swiss French | 1'234'567,89 | latn |  |
| **`fr-LU`** | French (Luxembourg) | **1.234.567,89** | latn | ◆ region `LU` |
| `fur` | Friulian | 1.234.567,89 | latn |  |
| `fy` | Western Frisian | 1.234.567,89 | latn |  |
| `ga` | Irish | 1,234,567.89 | latn |  |
| `gd` | Scottish Gaelic | 1,234,567.89 | latn |  |
| `gl` | Galician | 1.234.567,89 | latn |  |
| `gsw` | Swiss German | 1'234'567.89 | latn |  |
| `gv` | Manx | 1,234,567.89 | latn |  |
| **`hr`** | Croatian | **1.234.567,89** | latn | ★ lang `hr` |
| `hr-BA` | Croatian (Bosnia & Herzegovina) | 1.234.567,89 | latn |  |
| `hsb` | Upper Sorbian | 1.234.567,89 | latn |  |
| **`hu`** | Hungarian | **1 234 567,89** | latn | ★ lang `hu` |
| `is` | Icelandic | 1.234.567,89 | latn |  |
| **`it`** | Italian | **1.234.567,89** | latn | ★ lang `it` |
| `it-CH` | Italian (Switzerland) | 1'234'567.89 | latn |  |
| `it-SM` | Italian (San Marino) | 1.234.567,89 | latn |  |
| `it-VA` | Italian (Vatican City) | 1.234.567,89 | latn |  |
| `kw` | Cornish | 1,234,567.89 | latn |  |
| `lb` | Luxembourgish | 1.234.567,89 | latn |  |
| `lt` | Lithuanian | 1 234 567,89 | latn |  |
| `lv` | Latvian | 1 234 567,89 | latn |  |
| `mk` | Macedonian | 1.234.567,89 | latn |  |
| `mt` | Maltese | 1,234,567.89 | latn |  |
| **`nb`** | Norwegian Bokmål | **1 234 567,89** | latn | ★ lang `nb` |
| `nds` | Low German | 1.234.567,89 | latn |  |
| **`nl`** | Dutch | **1.234.567,89** | latn | ★ lang `nl` |
| **`nl-BE`** | Flemish | **1.234.567,89** | latn | ◆ region `BE` |
| `nn` | Norwegian Nynorsk | 1 234 567,89 | latn |  |
| `oc` | Occitan | 1 234 567,89 | latn |  |
| **`pl`** | Polish | **1 234 567,89** | latn | ★ lang `pl` |
| **`pt`** | Portuguese | **1.234.567,89** | latn | ★ lang `pt` |
| **`pt-PT`** | European Portuguese | **1 234 567,89** | latn | ◆ region `PT` |
| `rm` | Romansh | 1 234 567,89 | latn |  |
| **`ro`** | Romanian | **1.234.567,89** | latn | ★ lang `ro` |
| `ro-MD` | Moldavian | 1.234.567,89 | latn |  |
| **`ru`** | Russian | **1 234 567,89** | latn | ★ lang `ru` |
| `ru-BY` | Russian (Belarus) | 1 234 567,89 | latn |  |
| `ru-MD` | Russian (Moldova) | 1 234 567,89 | latn |  |
| `ru-UA` | Russian (Ukraine) | 1 234 567,89 | latn |  |
| `sc` | Sardinian | 1.234.567,89 | latn |  |
| `se` | Northern Sami | 1 234 567,89 | latn |  |
| `se-FI` | Northern Sami (Finland) | 1 234 567,89 | latn |  |
| `se-SE` | Northern Sami (Sweden) | 1 234 567,89 | latn |  |
| **`sk`** | Slovak | **1 234 567,89** | latn | ★ lang `sk` |
| `sl` | Slovenian | 1.234.567,89 | latn |  |
| `smn` | Inari Sami | 1 234 567,89 | latn |  |
| `sq` | Albanian | 1 234 567,89 | latn |  |
| `sq-MK` | Albanian (North Macedonia) | 1 234 567,89 | latn |  |
| `sq-XK` | Albanian (Kosovo) | 1 234 567,89 | latn |  |
| **`sr`** | Serbian | **1.234.567,89** | latn | ★ lang `sr` |
| `sr-Latn` | Serbian (Latin) | 1.234.567,89 | latn |  |
| **`sv`** | Swedish | **1 234 567,89** | latn | ★ lang `sv` |
| `sv-FI` | Swedish (Finland) | 1 234 567,89 | latn |  |
| `tr-CY` | Turkish (Cyprus) | 1.234.567,89 | latn |  |
| **`uk`** | Ukrainian | **1 234 567,89** | latn | ★ lang `uk` |
| `vec` | Venetian | 1 234 567,89 | latn |  |
| `wae` | Walser | 1'234'567,89 | latn |  |

### Middle East

| Locale | Language | Sample `1234567.89` | Digits | Default of |
|---|---|---|---|---|
| **`ar`** | Arabic | **1,234,567.89** | latn | ★ lang `ar` |
| **`ar-AE`** | Arabic (United Arab Emirates) | **1,234,567.89** | latn | ◆ region `AE` |
| **`ar-BH`** | Arabic (Bahrain) | **١٬٢٣٤٬٥٦٧٫٨٩** | arab | ◆ region `BH` |
| `ar-IL` | Arabic (Israel) | ١٬٢٣٤٬٥٦٧٫٨٩ | arab |  |
| **`ar-IQ`** | Arabic (Iraq) | **١٬٢٣٤٬٥٦٧٫٨٩** | arab | ◆ region `IQ` |
| **`ar-JO`** | Arabic (Jordan) | **١٬٢٣٤٬٥٦٧٫٨٩** | arab | ◆ region `JO` |
| **`ar-KW`** | Arabic (Kuwait) | **١٬٢٣٤٬٥٦٧٫٨٩** | arab | ◆ region `KW` |
| **`ar-LB`** | Arabic (Lebanon) | **١٬٢٣٤٬٥٦٧٫٨٩** | arab | ◆ region `LB` |
| **`ar-OM`** | Arabic (Oman) | **١٬٢٣٤٬٥٦٧٫٨٩** | arab | ◆ region `OM` |
| `ar-PS` | Arabic (Palestinian Territories) | ١٬٢٣٤٬٥٦٧٫٨٩ | arab |  |
| **`ar-QA`** | Arabic (Qatar) | **١٬٢٣٤٬٥٦٧٫٨٩** | arab | ◆ region `QA` |
| **`ar-SA`** | Arabic (Saudi Arabia) | **١٬٢٣٤٬٥٦٧٫٨٩** | arab | ◆ region `SA` |
| `ar-SY` | Arabic (Syria) | ١٬٢٣٤٬٥٦٧٫٨٩ | arab |  |
| `ar-YE` | Arabic (Yemen) | ١٬٢٣٤٬٥٦٧٫٨٩ | arab |  |
| `ckb` | Central Kurdish | ١٬٢٣٤٬٥٦٧٫٨٩ | arab |  |
| `ckb-IR` | Central Kurdish (Iran) | ١٬٢٣٤٬٥٦٧٫٨٩ | arab |  |
| **`fa`** | Persian | **۱٬۲۳۴٬۵۶۷٫۸۹** | arabext | ★ lang `fa` |
| `fa-AF` | Dari | ۱٬۲۳۴٬۵۶۷٫۸۹ | arabext |  |
| **`he`** | Hebrew | **1,234,567.89** | latn | ★ lang `he` |
| `syr` | Syriac | 1,234,567.89 | latn |  |
| **`tr`** | Turkish | **1.234.567,89** | latn | ★ lang `tr` |

### Oceania

| Locale | Language | Sample `1234567.89` | Digits | Default of |
|---|---|---|---|---|
| **`en-AU`** | Australian English | **1,234,567.89** | latn | ◆ region `AU` |
| **`en-NZ`** | English (New Zealand) | **1,234,567.89** | latn | ◆ region `NZ` |
| `fj` | Fijian | 1,234,567.89 | latn |  |
| `haw` | Hawaiian | 1,234,567.89 | latn |  |
| `mi` | Māori | 1,234,567.89 | latn |  |
| `to` | Tongan | 1,234,567.89 | latn |  |

### Other / generic

| Locale | Language | Sample `1234567.89` | Digits | Default of |
|---|---|---|---|---|
| **`en`** | English | **1,234,567.89** | latn | ★ lang `en` |
| `en-001` | English (world) | 1,234,567.89 | latn |  |
| `en-150` | English (Europe) | 1,234,567.89 | latn |  |
| `eo` | Esperanto | 1 234 567,89 | latn |  |
| **`fr-FR`** | French (France) | **1 234 567,89** | latn | ★ lang `fr` · ◆ region `FR` |
| **`de-DE`** | German (Germany) | **1.234.567,89** | latn | ★ lang `de` · ◆ region `DE` |
| **`es`** | Spanish | **1.234.567,89** | latn | ★ lang `es` |
| **`it-IT`** | Italian (Italy) | **1.234.567,89** | latn | ★ lang `it` · ◆ region `IT` |
| **`pt-BR`** | Brazilian Portuguese | **1.234.567,89** | latn | ★ lang `pt` · ◆ region `BR` |
| **`ru-RU`** | Russian (Russia) | **1 234 567,89** | latn | ★ lang `ru` |
| **`zh-CN`** | Chinese (China) | **1,234,567.89** | latn | ★ lang `zh-Hans` |
| **`zh-TW`** | Chinese (Taiwan) | **1,234,567.89** | latn | ★ lang `zh-Hant` · ◆ region `TW` |
| **`ja-JP`** | Japanese (Japan) | **1,234,567.89** | latn | ★ lang `ja` · ◆ region `JP` |
| **`ko-KR`** | Korean (South Korea) | **1,234,567.89** | latn | ★ lang `ko` · ◆ region `KR` |
| `ace` | Acehnese | 1,234,567.89 | latn |  |
| `ast` | Asturian | 1.234.567,89 | latn |  |
| `mg-MG` | Malagasy (Madagascar) | 1,234,567.89 | latn |  |
| `ia` | Interlingua | 1.234.567,89 | latn |  |
| `jbo` | Lojban | 1,234,567.89 | latn |  |
| `tlh` | Klingon | 1,234,567.89 | latn |  |

## Appendix C — Language → default locale (fallback step 2)

When only the language is usable, formatting follows the language's default locale, computed at runtime by `new Intl.Locale(lang).maximize()` (CLDR likely-subtags). Do not hardcode this table.

| App language | Name | Default locale | Default region | Sample | Digits |
|---|---|---|---|---|---|
| `en` | English | **`en-US`** | United States | 1,234,567.89 | latn |
| `es` | Spanish | **`es-ES`** | Spain | 1.234.567,89 | latn |
| `fr` | French | **`fr-FR`** | France | 1 234 567,89 | latn |
| `de` | German | **`de-DE`** | Germany | 1.234.567,89 | latn |
| `it` | Italian | **`it-IT`** | Italy | 1.234.567,89 | latn |
| `nl` | Dutch | **`nl-NL`** | Netherlands | 1.234.567,89 | latn |
| `sv` | Swedish | **`sv-SE`** | Sweden | 1 234 567,89 | latn |
| `da` | Danish | **`da-DK`** | Denmark | 1.234.567,89 | latn |
| `nb` | Norwegian Bokmål | **`nb-NO`** | Norway | 1 234 567,89 | latn |
| `fi` | Finnish | **`fi-FI`** | Finland | 1 234 567,89 | latn |
| `tr` | Turkish | **`tr-TR`** | Türkiye | 1.234.567,89 | latn |
| `id` | Indonesian | **`id-ID`** | Indonesia | 1.234.567,89 | latn |
| `ms` | Malay | **`ms-MY`** | Malaysia | 1,234,567.89 | latn |
| `fil` | Filipino | **`fil-PH`** | Philippines | 1,234,567.89 | latn |
| `pt` | Portuguese | **`pt-BR`** | Brazil | 1.234.567,89 | latn |
| `ru` | Russian | **`ru-RU`** | Russia | 1 234 567,89 | latn |
| `uk` | Ukrainian | **`uk-UA`** | Ukraine | 1 234 567,89 | latn |
| `pl` | Polish | **`pl-PL`** | Poland | 1 234 567,89 | latn |
| `cs` | Czech | **`cs-CZ`** | Czechia | 1 234 567,89 | latn |
| `sk` | Slovak | **`sk-SK`** | Slovakia | 1 234 567,89 | latn |
| `hu` | Hungarian | **`hu-HU`** | Hungary | 1 234 567,89 | latn |
| `ro` | Romanian | **`ro-RO`** | Romania | 1.234.567,89 | latn |
| `bg` | Bulgarian | **`bg-BG`** | Bulgaria | 1 234 567,89 | latn |
| `el` | Greek | **`el-GR`** | Greece | 1.234.567,89 | latn |
| `hr` | Croatian | **`hr-HR`** | Croatia | 1.234.567,89 | latn |
| `sr` | Serbian | **`sr-RS`** | Serbia | 1.234.567,89 | latn |
| `hi` | Hindi | **`hi-IN`** | India | 12,34,567.89 | latn |
| `bn` | Bangla | **`bn-BD`** | Bangladesh | ১২,৩৪,৫৬৭.৮৯ | beng |
| `gu` | Gujarati | **`gu-IN`** | India | 12,34,567.89 | latn |
| `pa` | Punjabi | **`pa-IN`** | India | 12,34,567.89 | latn |
| `ta` | Tamil | **`ta-IN`** | India | 12,34,567.89 | latn |
| `te` | Telugu | **`te-IN`** | India | 12,34,567.89 | latn |
| `kn` | Kannada | **`kn-IN`** | India | 1,234,567.89 | latn |
| `ml` | Malayalam | **`ml-IN`** | India | 12,34,567.89 | latn |
| `ur` | Urdu | **`ur-PK`** | Pakistan | 1,234,567.89 | latn |
| `fa` | Persian | **`fa-IR`** | Iran | ۱٬۲۳۴٬۵۶۷٫۸۹ | arabext |
| `he` | Hebrew | **`he-IL`** | Israel | 1,234,567.89 | latn |
| `ar` | Arabic | **`ar-EG`** | Egypt | 1,234,567.89 | latn |
| `th` | Thai | **`th-TH`** | Thailand | 1,234,567.89 | latn |
| `vi` | Vietnamese | **`vi-VN`** | Vietnam | 1.234.567,89 | latn |
| `km` | Khmer | **`km-KH`** | Cambodia | 1,234,567.89 | latn |
| `my` | Burmese | **`my-MM`** | Myanmar (Burma) | ၁,၂၃၄,၅၆၇.၈၉ | mymr |
| `ja` | Japanese | **`ja-JP`** | Japan | 1,234,567.89 | latn |
| `ko` | Korean | **`ko-KR`** | South Korea | 1,234,567.89 | latn |
| `mr` | Marathi | **`mr-IN`** | India | १२,३४,५६७.८९ | deva |
| `zh-Hant` | Traditional Chinese | **`zh-Hant-TW`** | Taiwan | 1,234,567.89 | latn |
| `zh-Hans` | Simplified Chinese | **`zh-Hans-CN`** | China | 1,234,567.89 | latn |

## Appendix D — Region → default locale (fallback step 3)

When the language is unusable, formatting follows the region's default locale, computed at runtime by `new Intl.Locale("und-"+REGION).maximize()`. Do not hardcode this table.

| Region | Country | Default language | Default locale | Sample | Digits |
|---|---|---|---|---|---|
| `US` | United States | English (`en`) | **`en-US`** | 1,234,567.89 | latn |
| `CA` | Canada | English (`en`) | **`en-CA`** | 1,234,567.89 | latn |
| `MX` | Mexico | Spanish (`es`) | **`es-MX`** | 1,234,567.89 | latn |
| `BR` | Brazil | Portuguese (`pt`) | **`pt-BR`** | 1.234.567,89 | latn |
| `AR` | Argentina | Spanish (`es`) | **`es-AR`** | 1.234.567,89 | latn |
| `CL` | Chile | Spanish (`es`) | **`es-CL`** | 1.234.567,89 | latn |
| `CO` | Colombia | Spanish (`es`) | **`es-CO`** | 1.234.567,89 | latn |
| `PE` | Peru | Spanish (`es`) | **`es-PE`** | 1,234,567.89 | latn |
| `EC` | Ecuador | Spanish (`es`) | **`es-EC`** | 1.234.567,89 | latn |
| `GT` | Guatemala | Spanish (`es`) | **`es-GT`** | 1,234,567.89 | latn |
| `DO` | Dominican Republic | Spanish (`es`) | **`es-DO`** | 1,234,567.89 | latn |
| `CR` | Costa Rica | Spanish (`es`) | **`es-CR`** | 1 234 567,89 | latn |
| `PA` | Panama | Spanish (`es`) | **`es-PA`** | 1,234,567.89 | latn |
| `UY` | Uruguay | Spanish (`es`) | **`es-UY`** | 1.234.567,89 | latn |
| `BO` | Bolivia | Spanish (`es`) | **`es-BO`** | 1.234.567,89 | latn |
| `PY` | Paraguay | Guarani (`gn`) | **`gn-PY`** | 1,234,567.89 | latn |
| `GB` | United Kingdom | English (`en`) | **`en-GB`** | 1,234,567.89 | latn |
| `IE` | Ireland | English (`en`) | **`en-IE`** | 1,234,567.89 | latn |
| `FR` | France | French (`fr`) | **`fr-FR`** | 1 234 567,89 | latn |
| `DE` | Germany | German (`de`) | **`de-DE`** | 1.234.567,89 | latn |
| `ES` | Spain | Spanish (`es`) | **`es-ES`** | 1.234.567,89 | latn |
| `IT` | Italy | Italian (`it`) | **`it-IT`** | 1.234.567,89 | latn |
| `PT` | Portugal | Portuguese (`pt`) | **`pt-PT`** | 1 234 567,89 | latn |
| `NL` | Netherlands | Dutch (`nl`) | **`nl-NL`** | 1.234.567,89 | latn |
| `BE` | Belgium | Dutch (`nl`) | **`nl-BE`** | 1.234.567,89 | latn |
| `LU` | Luxembourg | French (`fr`) | **`fr-LU`** | 1.234.567,89 | latn |
| `CH` | Switzerland | German (`de`) | **`de-CH`** | 1'234'567.89 | latn |
| `AT` | Austria | German (`de`) | **`de-AT`** | 1 234 567,89 | latn |
| `SE` | Sweden | Swedish (`sv`) | **`sv-SE`** | 1 234 567,89 | latn |
| `NO` | Norway | Norwegian Bokmål (`nb`) | **`nb-NO`** | 1 234 567,89 | latn |
| `DK` | Denmark | Danish (`da`) | **`da-DK`** | 1.234.567,89 | latn |
| `FI` | Finland | Finnish (`fi`) | **`fi-FI`** | 1 234 567,89 | latn |
| `IS` | Iceland | Icelandic (`is`) | **`is-IS`** | 1.234.567,89 | latn |
| `PL` | Poland | Polish (`pl`) | **`pl-PL`** | 1 234 567,89 | latn |
| `CZ` | Czechia | Czech (`cs`) | **`cs-CZ`** | 1 234 567,89 | latn |
| `SK` | Slovakia | Slovak (`sk`) | **`sk-SK`** | 1 234 567,89 | latn |
| `HU` | Hungary | Hungarian (`hu`) | **`hu-HU`** | 1 234 567,89 | latn |
| `RO` | Romania | Romanian (`ro`) | **`ro-RO`** | 1.234.567,89 | latn |
| `BG` | Bulgaria | Bulgarian (`bg`) | **`bg-BG`** | 1 234 567,89 | latn |
| `GR` | Greece | Greek (`el`) | **`el-GR`** | 1.234.567,89 | latn |
| `HR` | Croatia | Croatian (`hr`) | **`hr-HR`** | 1.234.567,89 | latn |
| `SI` | Slovenia | Slovenian (`sl`) | **`sl-SI`** | 1.234.567,89 | latn |
| `RS` | Serbia | Serbian (`sr`) | **`sr-RS`** | 1.234.567,89 | latn |
| `UA` | Ukraine | Ukrainian (`uk`) | **`uk-UA`** | 1 234 567,89 | latn |
| `LT` | Lithuania | Lithuanian (`lt`) | **`lt-LT`** | 1 234 567,89 | latn |
| `LV` | Latvia | Latvian (`lv`) | **`lv-LV`** | 1 234 567,89 | latn |
| `EE` | Estonia | Estonian (`et`) | **`et-EE`** | 1 234 567,89 | latn |
| `AE` | United Arab Emirates | Arabic (`ar`) | **`ar-AE`** | 1,234,567.89 | latn |
| `SA` | Saudi Arabia | Arabic (`ar`) | **`ar-SA`** | ١٬٢٣٤٬٥٦٧٫٨٩ | arab |
| `QA` | Qatar | Arabic (`ar`) | **`ar-QA`** | ١٬٢٣٤٬٥٦٧٫٨٩ | arab |
| `KW` | Kuwait | Arabic (`ar`) | **`ar-KW`** | ١٬٢٣٤٬٥٦٧٫٨٩ | arab |
| `BH` | Bahrain | Arabic (`ar`) | **`ar-BH`** | ١٬٢٣٤٬٥٦٧٫٨٩ | arab |
| `OM` | Oman | Arabic (`ar`) | **`ar-OM`** | ١٬٢٣٤٬٥٦٧٫٨٩ | arab |
| `EG` | Egypt | Arabic (`ar`) | **`ar-EG`** | ١٬٢٣٤٬٥٦٧٫٨٩ | arab |
| `JO` | Jordan | Arabic (`ar`) | **`ar-JO`** | ١٬٢٣٤٬٥٦٧٫٨٩ | arab |
| `LB` | Lebanon | Arabic (`ar`) | **`ar-LB`** | ١٬٢٣٤٬٥٦٧٫٨٩ | arab |
| `IQ` | Iraq | Arabic (`ar`) | **`ar-IQ`** | ١٬٢٣٤٬٥٦٧٫٨٩ | arab |
| `MA` | Morocco | Arabic (`ar`) | **`ar-MA`** | 1.234.567,89 | latn |
| `DZ` | Algeria | Arabic (`ar`) | **`ar-DZ`** | 1.234.567,89 | latn |
| `TN` | Tunisia | Arabic (`ar`) | **`ar-TN`** | 1.234.567,89 | latn |
| `IL` | Israel | Hebrew (`he`) | **`he-IL`** | 1,234,567.89 | latn |
| `TR` | Türkiye | Turkish (`tr`) | **`tr-TR`** | 1.234.567,89 | latn |
| `NG` | Nigeria | English (`en`) | **`en-NG`** | 1,234,567.89 | latn |
| `KE` | Kenya | Swahili (`sw`) | **`sw-KE`** | 1,234,567.89 | latn |
| `GH` | Ghana | Akan (`ak`) | **`ak-GH`** | 1,234,567.89 | latn |
| `ZA` | South Africa | English (`en`) | **`en-ZA`** | 1 234 567,89 | latn |
| `TZ` | Tanzania | Swahili (`sw`) | **`sw-TZ`** | 1,234,567.89 | latn |
| `UG` | Uganda | Swahili (`sw`) | **`sw-UG`** | 1,234,567.89 | latn |
| `SN` | Senegal | Wolof (`wo`) | **`wo-SN`** | 1.234.567,89 | latn |
| `CI` | Côte d’Ivoire | French (`fr`) | **`fr-CI`** | 1 234 567,89 | latn |
| `ET` | Ethiopia | Amharic (`am`) | **`am-ET`** | 1,234,567.89 | latn |
| `PK` | Pakistan | Urdu (`ur`) | **`ur-PK`** | 1,234,567.89 | latn |
| `BD` | Bangladesh | Bangla (`bn`) | **`bn-BD`** | ১২,৩৪,৫৬৭.৮৯ | beng |
| `LK` | Sri Lanka | Sinhala (`si`) | **`si-LK`** | 1,234,567.89 | latn |
| `NP` | Nepal | Nepali (`ne`) | **`ne-NP`** | १२,३४,५६७.८९ | deva |
| `KZ` | Kazakhstan | Russian (`ru`) | **`ru-KZ`** | 1 234 567,89 | latn |
| `UZ` | Uzbekistan | Uzbek (`uz`) | **`uz-UZ`** | 1 234 567,89 | latn |
| `SG` | Singapore | English (`en`) | **`en-SG`** | 1,234,567.89 | latn |
| `MY` | Malaysia | Malay (`ms`) | **`ms-MY`** | 1,234,567.89 | latn |
| `ID` | Indonesia | Indonesian (`id`) | **`id-ID`** | 1.234.567,89 | latn |
| `TH` | Thailand | Thai (`th`) | **`th-TH`** | 1,234,567.89 | latn |
| `VN` | Vietnam | Vietnamese (`vi`) | **`vi-VN`** | 1.234.567,89 | latn |
| `PH` | Philippines | Filipino (`fil`) | **`fil-PH`** | 1,234,567.89 | latn |
| `KH` | Cambodia | Khmer (`km`) | **`km-KH`** | 1,234,567.89 | latn |
| `MM` | Myanmar (Burma) | Burmese (`my`) | **`my-MM`** | ၁,၂၃၄,၅၆၇.၈၉ | mymr |
| `LA` | Laos | Lao (`lo`) | **`lo-LA`** | 1.234.567,89 | latn |
| `BN` | Brunei | Malay (`ms`) | **`ms-BN`** | 1.234.567,89 | latn |
| `JP` | Japan | Japanese (`ja`) | **`ja-JP`** | 1,234,567.89 | latn |
| `KR` | South Korea | Korean (`ko`) | **`ko-KR`** | 1,234,567.89 | latn |
| `TW` | Taiwan | Chinese (`zh`) | **`zh-TW`** | 1,234,567.89 | latn |
| `HK` | Hong Kong SAR China | Chinese (`zh`) | **`zh-HK`** | 1,234,567.89 | latn |
| `MO` | Macao SAR China | Chinese (`zh`) | **`zh-MO`** | 1,234,567.89 | latn |
| `AU` | Australia | English (`en`) | **`en-AU`** | 1,234,567.89 | latn |
| `NZ` | New Zealand | English (`en`) | **`en-NZ`** | 1,234,567.89 | latn |
| `FJ` | Fiji | English (`en`) | **`en-FJ`** | 1,234,567.89 | latn |
| `IN` | India | Hindi (`hi`) | **`hi-IN`** | 12,34,567.89 | latn |
