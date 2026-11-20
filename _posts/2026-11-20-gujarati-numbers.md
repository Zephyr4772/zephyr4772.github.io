---
layout: post
title: "Half past nine, ₹1.5 lakh, ₹1 crore: where Indian-language AI gets numbers wrong"
description: "As a native Gujarati speaker, I tested how Sarvam's voices, its spoken-form API, a shared number-to-words library and IndicTrans2 handle written numbers. Half the Gujarati sentences with a number came out wrong, a clock time of 9:30 became 10:15, and a translation turned ₹1 crore into ₹1,000. Fixes for all of them are open source."
tag: Evaluation
image: /assets/img/gujarati-numbers/cover.png
image_alt: "Sarvam Bulbul TTS on 108 Gujarati sentences: 59 misread on bulbul:v3 before a text fix, 0 after. An IndicTrans2 translation turned 1 crore rupees into 1,000. Examples: 9:30 read as nine o'clock thirty minutes o'clock instead of half past nine; 1.5 lakh rupees read with the English word rupees; 10-15 days read as one zero one five days."
---

Indian languages say numbers in ways English doesn't. In Gujarati, 9:30 is *sāḍā nav* ("nine and a half"), 1.5 lakh is *doḍh lākh*, ₹99.50 is "ninety-nine rupees fifty paise", and 1,00,00,000 is one crore, written with Indian digit grouping. Any system that turns written text into speech, or translates it, has to get these right. Money, times and phone numbers are exactly what a bank or government voice agent says most.

I'm a native Gujarati speaker, so I spent two days testing how well today's Indian-language AI does this: Sarvam's Bulbul voices and its text APIs, the `indic-numtowords` library that Sarvam and AI4Bharat both use, and AI4Bharat's IndicTrans2 translation models.

**The short version:**

- **Sarvam's Bulbul voices misread half the Gujarati sentences that contain a number:** 59, 55 and 56 of 108 on three voices. Every ₹ amount, every clock time that isn't a whole hour, and every "and a half" decimal came out wrong. The voices themselves are fine: the same sentences written out in words are spoken correctly. The fault is in the text cleanup before speech.
- **A 0.13 ms text-cleanup step fixes it:** 59 / 55 / 56 → **0 / 2 / 1** misread. In a blind listening test I judged Sarvam's reading correct 0 of 20 times and the fixed one 19 of 20, and preferred the fixed one 18 to 1 (p = 0.00008).
- **Sarvam's own `spoken_form` API gets Gujarati times wrong:** 7 of 10, against 2 of 10 for Hindi. 9:30 becomes *savā das*, which means 10:15.
- **IndicTrans2 can silently change an amount:** "Rs 1,00,00,000" came out as "₹1,000", and "Rs 12,34,567 lakh" as "₹1 lakh". A 20-line fix to the toolkit cuts split numbers from 26 to 3 per model, with no new errors.

## How I tested speech

I wrote 177 Gujarati sentences that stress numbers: times, money, decimals, ranges, ordinals, dates, units and IDs. Each went through Sarvam's text-to-speech (Bulbul v3 and two v4-flash voices), then back through Sarvam's own speech-to-text (Saaras v4, verbatim mode), and I checked whether the expected spoken words were there.

Two controls keep this honest. The same sentences written out in words by hand are spoken correctly, so a misreading means the text cleanup failed, not the voice. And the speech-to-text judge is checked by a blind human listening test, below.

## What Bulbul says

| Written | What Bulbul says | Correct |
|---|---|---|
| સવારે 9:30 વાગ્યે | "nine o'clock thirty minutes o'clock" (ungrammatical) | સાડા નવ વાગ્યે ("half past nine") |
| લોન ₹1.5 લાખની | "one decimal five lakh **rupees**", in English | દોઢ લાખ રૂપિયાની |
| 2.5 કિલો | "two **point** five kilo" | અઢી કિલો |
| ₹99.50 | "ninety-nine decimal five rupees" (the paise are lost) | નવ્વાણું રૂપિયા પચાસ પૈસા |
| 10-15 દિવસ | "**one zero one five** days" | દસથી પંદર દિવસ |
| 1લા નંબરે | એકલા, which means "alone" | પહેલા ("first") |
| 8.5% | "eight decimal five **percent**" | સાડા આઠ ટકા |

How often, over the 108 sentences that contain a number or symbol:

| Pattern | v3 | v4 `pooja` | v4 `bhavik` |
|---|---|---|---|
| ₹ / Rs. read as the English word "rupees" | 20/20 | 20/20 | 20/20 |
| Clock time that isn't a whole hour | 9/9 | 9/9 | 9/9 |
| x.5 / x.25 / x.75 not read as દોઢ / અઢી / સાડા / સવા / પોણા | 13/13 | 13/13 | 13/13 |
| **Any misreading** | **59 (55%)** | **55 (51%)** | **56 (52%)** |

53 sentences fail on all three voices. It's the same with Sarvam's `enable_preprocessing` option on or off. Hindi does better on ₹, years and times, but has the same decimal, range and ordinal problems.

It compounds in a voice agent. I ran `sarvam-105b-conversations` as a Gujarati bank agent: it wrote `9:30`, `10.75%` and `₹` in 8 to 11 of 12 replies, and Bulbul v4 then spoke "the branch opens at 9:30" as "nine o'clock thirty minutes o'clock". Sarvam retired Bulbul v3 on Voice Agents on 10 November, so every Gujarati agent now runs on v4.

## Sarvam's spoken-form API gets times wrong

Sarvam's Transliterate API has a `spoken_form` option that is meant to do this cleanup. For Gujarati clock times it's wrong 7 times out of 10, and the same on every run:

| Input | Output | What it means | Hindi |
|---|---|---|---|
| 9:30 | સવા દસ | **10:15** | correct |
| 2:30 | બસ્ઠી | not a word | correct |
| 6:45 | સાડા છ | **6:30** | wrong |
| `9:30am`, the docs' own example | સવારે સવા દસ વાગ્યે એ.એમ. | **10:15**, with "AM" said twice | correct |

## The fix: gujnorm

[gujnorm](https://github.com/Zephyr4772/gujnorm) is a deterministic text-to-text step that runs before TTS. It's pure Python with no dependencies, takes about 0.13 ms per sentence, and has 100 unit tests. It covers times (સવા, સાડા, પોણા, દોઢ, અઢી, and AM/PM as સવારે/સાંજે), money with paise, lakh and crore, percentages, ranges, ordinals, dates and years, and reads phone, OTP and account numbers digit by digit. Alphanumeric IDs are left alone.

| Voice | Misread: before → after (of 108) | English-word slips |
|---|---|---|
| bulbul:v3 | 59 → **0** | 15 → 2 |
| v4-flash `pooja_gu_customer` | 55 → **2** | 15 → 2 |
| v4-flash `bhavik_gu_conversation` | 56 → **1** | 11 → 1 |

The 3 that remain are on the voice side: v4 says પોણો in a way the speech-to-text hears as a different word. On Sarvam's own `tts-robustness-benchmark` (81 Gujarati sentences), gujnorm rewrites 35, including 21 of the 23 in its numerics category, and leaves no stray digits.

**Blind listening test.** I listened to 36 random before/after pairs from the three voices, with the clip order hidden and the answer key kept out of the page until scoring:

| Pairs | Judged correct: Sarvam → gujnorm | Preferred: gujnorm / Sarvam / same | Sign test |
|---|---|---|---|
| Sarvam misread the sentence (20) | **0 → 19** | **18 / 1 / 1** | p = 0.00008 |
| Same words in both clips (16) | 12 → 7 | 4 / 9 / 3 | p = 0.27, no real difference |

The second row matters. When gujnorm didn't change the words, I didn't prefer it, so the first row is about the words, not a bias towards one version. Ten before/after clips are [in the repo](https://github.com/Zephyr4772/gujnorm/tree/main/samples), and there's a [live demo](https://huggingface.co/spaces/sinhal/gujnorm) that runs in the browser.

I sent the numbers to Sarvam support, and opened [sarvamai/skills #47](https://github.com/sarvamai/skills/pull/47), which adds the Gujarati number gotcha to their text-to-speech skill so developers and coding agents normalize numbers before calling Bulbul.

## The shared number-to-words library

Sarvam's [`indic-numtowords`](https://github.com/sarvamai/indic-numtowords) is a fork of AI4Bharat's, and both copies have the same bugs in their Gujarati and Hindi tables:

- **Hindi "सावा" instead of "सवा"** in 9 entries, from 125 ("सवा सौ") to 925.
- **Gujarati crore written in Punjabi script:** `"10000000": ["ਕਰੋੜ"]` is Gurmukhi, not કરોડ. It's the only wrong-script word in all the tables.
- **No Gujarati variations** like સવાસો (125), દોઢસો (150) and અઢીસો (250), which Hindi already has.

The fix leaves the default output unchanged for every number from 0 to 100,000 in Gujarati and Hindi: 0 differences in 200,002 outputs. 4 new tests fail before it and pass after. PRs: [sarvamai #5](https://github.com/sarvamai/indic-numtowords/pull/5) and [AI4Bharat #16](https://github.com/AI4Bharat/indic-numtowords/pull/16).

## IndicTrans2: amounts that change in translation

IndicTrans2 is AI4Bharat's open translation model for Indian languages. Its toolkit replaces numbers with placeholders before translation, so the model copies them unchanged. I translated 255 English sentences with real banking, government and health numbers into Gujarati and Hindi, with the 200M and 1B models through the official pipeline.

Indian-grouped amounts were only partly masked, so the model saw half a number:

| Source | After pre-processing | Gujarati output |
|---|---|---|
| ₹7,302.34 | `₹ 7 , <ID1>` | ₹7, 302.34 |
| ₹1,37,11,477 | `₹ <ID1> , 477` | ₹ 1,37,11, 477 |
| Rs 1,00,00,000 | `Rs <ID1> , 000` | **₹1,000** |

There were three causes:
1. A URL pattern ran first and took "302.34" for a domain name.
2. The number pattern only covered three comma groups, and crore amounts have four or more.
3. Replacing one number could also change it inside a longer one, depending on the order.

[PR #32](https://github.com/VarunGumma/IndicTransToolkit/pull/32) is about 20 lines and 8 tests:

| Model → language | Split numbers | Fully correct sentences | Newly wrong |
|---|---|---|---|
| 200M → Gujarati | 26 → **3** | 207 → **230** | **0** |
| 200M → Hindi | 27 → **4** | 206 → **229** | **0** |
| 1B → Gujarati | 27 → **4** | 201 → **224** | **0** |

I left one case alone on purpose: "8.5%" still comes out as "8. 5". Masking short decimals too made "Rs 3.75 lakh" come out as "₹1 lakh", and a cosmetic split is much better than a wrong amount.

That points to a second, separate problem. When the model writes a placeholder without its brackets, post-processing can't restore it, and the amount becomes the placeholder's serial number: "revised to **Rs 12,34,567 lakh**" came out as "**₹1 lakh**". In a stress test of 150 sentences in each language, 23 of 300 values were lost on the current release, and 18 with the PR. All 18 are "Rs N lakh/crore" phrasing and happen on the current release too. I haven't reported it yet.

## What I'd check in any Indian-language voice or translation product

1. **Test with the numbers your users actually say.** Money with paise, half-past times, lakh and crore, ranges and ordinals. A general benchmark won't catch these: the sentences need to be written to stress them.
2. **Normalize text before TTS, deterministically.** It's cheap, it's testable, and it's where these errors come from.
3. **Never trust a translated amount without checking it.** Compare the numbers in the source and the output after normalizing the digits. A split number is ugly; a changed one is dangerous.
4. **Separate the voice from the text cleanup.** Write the same sentence out in words. If that's spoken correctly, the voice is fine and the fix is upstream.

## Method and limits

- **Judges:** for audio, Saaras v4 verbatim speech-to-text, with digits in the transcript normalized the same way for both clips, plus the blind listening test. For translation, exact number matching after digit normalization.
- **Coverage:** I wrote and checked the expected readings myself, as one native speaker. The sentences were written to stress numbers, so ordinary text will show lower error rates.
- **Dates:** results reflect the APIs and models as of November 2026.
- **Not done yet:** the full audio benchmark on Sarvam's set, and an audio re-run of the latest gujnorm release. I ran out of API credits.

Everything is public: the [gujnorm code, tests and samples](https://github.com/Zephyr4772/gujnorm), the [TTS dataset](https://huggingface.co/datasets/sinhal/gujarati-tts-number-reading) with 324 before/after results and the listening test, the [IndicTrans2 dataset](https://huggingface.co/datasets/sinhal/indictrans2-number-fidelity) with 1,530 translations, and the [demo](https://huggingface.co/spaces/sinhal/gujnorm).

## Status

| What | Status |
|---|---|
| [sarvamai/indic-numtowords #5](https://github.com/sarvamai/indic-numtowords/pull/5) | open |
| [sarvamai/skills #47](https://github.com/sarvamai/skills/pull/47) | open |
| [AI4Bharat/indic-numtowords #16](https://github.com/AI4Bharat/indic-numtowords/pull/16) | open |
| [VarunGumma/IndicTransToolkit #32](https://github.com/VarunGumma/IndicTransToolkit/pull/32) | open |
| Bug report to Sarvam support | sent 19 November |
