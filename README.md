# -GSoC2026 FINAL REPORT-
Google Summer of Code 2026 - Generate l10n file on per locale basis - OWHF

| | |
|---|---|
| **Contributor** | Aaron Chewe |
| **Organisation** | Python Software Foundation -Open World Holidays Framework |
| **Project** | [Generate l10n file on per locale basis](https://github.com/vacanza/holidays/issues/1658) |
| **Pull Request** | [#3585](https://github.com/vacanza/holidays/pull/3585) |
| **Mentors** | Serhii Murza, PPsyrius, Ark Yakovets, Kate Golovanova|
| **Period** | May 25 – July 27, 2026 |

---
##  Project Goal
The holidays library supports translations for over 175 countries. Before this project, every country had its own separate translation file for every language it supported ,producing over 666 individual files. This made managing translations difficult: updating a shared holiday name like "Christmas Day" required touching dozens of files.

The goal of this project was to migrate to a simpler model, one translation file per language, containing all holidays from all countries. Instead of holidays/locale/bg/LC_MESSAGES/BG.mo (Bulgarian translations for Bulgaria), the new system produces holidays/locale/bg.mo (Bulgarian translations for every holiday in the library). This makes the translation system easier to manage and easier to contribute to.

### Step 1 — Building the Foundation: The Intermediate JSON
The first challenge was that the library had 666 existing .po translation files, each containing holiday names for one country in one language. Before generating the new per-locale files, all of this data needed to be collected, organized, and verified in one place.
### What was built
A script called json_builder.py reads all 666 existing .po files and consolidates their content into a single structured file called holidays_l10n.json. Each entry in this JSON represents one holiday and contains its English name (used as the translation key), all its translations across every language, and the list of countries it applies to.

For example, Hong Kong's Lunar New Year's Eve entry looks like this:

<img width="1409" height="696" alt="image" src="https://github.com/user-attachments/assets/9718791e-de0e-4a2c-aa2f-58136b540200" />


This entry is separate from the general Chinese New Year's Eve entry because Hong Kong's government uses a legally distinct official name — confirmed by researching the Cap. 149 General Holidays Ordinance. This is an example of the linguistic research that was a significant part of Step 1

### The splitting mechanism
A key feature of json_builder.py is the ability to split a general holiday entry into a country-specific one when that country has a genuinely different official name. For example, Bulgaria's Independence Day uses a different name than the 92 other countries that also have an "Independence Day" holiday. Running

<img width="751" height="84" alt="image" src="https://github.com/user-attachments/assets/c869775c-3ce4-4595-96d8-e4fb07c18a05" />

Creates a separate entry for Bulgaria while correctly removing Bulgaria's translations from the general entry. A significant engineering challenge was ensuring that language-country codes like en_HK and zh_HK were handled correctly during splits ,a local translation variant is still valid for the general entry even after the country is split out.

### Linguistic research work
Step 1 required not just coding but careful linguistic verification. Key findings included:
- Hong Kong Lunar New Year's Eve — researched Cap. 149 General Holidays Ordinance to confirm HK uses a descriptive phrase because this day is not a statutory holiday in HK the way it is in China, Macau and Taiwan
- UAE Thai localization merged  - while analyzing the JSON, I identified inconsistencies in how Thai translations were being generated for UAE holidays. My mentors saw this in the PR review and fixed it immediately in [PR #3596](https://github.com/vacanza/holidays/pull/3596), which was merged in reference to my PR, showing how effective the project is at early stages.**
- Monaco French translations merged — the intermediate JSON surfaced that Monaco's Labor Day was stored as "Fête de la Travaille". The word Travaille is a verb form in French meaning "he/she works" grammatically wrong as a noun. The correct term is "Fête du Travail". This finding led my mentor to do a comprehensive Monaco update in [PR #3678](https://github.com/vacanza/holidays/pull/3678), going beyond just fixing the typo to creating a dedicated fr_MC locale with properly formatted translations following Monaco's official naming convention

---
### Step 2 — Generating the New Per-Locale Files
Once the JSON was built and verified, the next step was generating the actual translation files in the new format.
### What was built
A script called generate_locale_po_files.py reads holidays_l10n.json and produces:
- One master template file (holidays.pot) containing all holiday msgids
One .po file per language (bg.po, ar.po, uk.po etc.) — 160 files total — each containing every holiday in the library with that language's translation as the msgstr
-Every language file contains the complete set of 2452 entries. For holidays that don't have a translation in that language, the msgstr is left empty,this is standard Gettext practice and the runtime falls back to the English msgid automatically.
-The new_comment field added during linguistic review appears as the Gettext #. comment in the generated files, giving translators meaningful context rather than just the generic original comment.

---
### step 3 — Updating the Country Files
With the new translation system in place, the country .py files needed updating. Currently each file calls tr("Holiday name in local language") for example Bulgaria calls tr("Ден на Независимостта на България"). In the new system, tr() should use the English msgid instead: tr("Independence Day of Bulgaria")
### What was built
A script called replace_tr_strings.py automatically updates all these calls across every country and financial market file in the library. It uses Python's Abstract Syntax Tree (AST) module to parse each file and make precise replacements unlike simple string search-and-replace, this approach cannot accidentally break code by merging lines or dropping closing parentheses.
### Key challenges solved:
- Multi-line strings — some tr() calls span multiple lines with concatenated strings in different character sets. The script handles these correctly using byte-to-character offset conversion for multibyte characters like Cyrillic, Arabic and Chinese
- Ambiguous strings — some translation strings appear in multiple different holidays. The script identifies these and skips them rather than making a wrong replacement
- Comment cleanup — old l10n comments above each tr() call are updated with the new_comment from the JSON, or removed if no new comment exists
The script identified 3557 replacements across all country and financial files

---
### What Is Still In Progress
### JSON — Nested Translation Variants
- Before the per-locale .po files can be generated cleanly, every language entry in the JSON must be a flat string  not a nested dictionary of country variants. For example, an entry like this blocks the generator:
  
<img width="963" height="316" alt="image" src="https://github.com/user-attachments/assets/439749a0-e071-4819-9e1d-1eee5643700f" />


- This means Côte d'Ivoire uses a different French name for All Saints' Day than Belgium and France. The generator cannot put two different French translations into one fr.po file for the same msgid — Gettext doesn't work that way. The solution is to split Côte d'Ivoire into its own entry with its own msgid, leaving the majority in the source entry with a flat fr: "Toussaint".
- This splitting work is also dependent on [PR # 3720] [https://github.com/vacanza/holidays/pull/3720]  "Update l10n: general unification"  which standardizes holiday name translations across countries first. Some entries that currently appear as different translations are actually just capitalisation inconsistencies that should be unified rather than split. Once PR #3720 merges, the JSON will be rebuilt from the unified .po files and the splits redone correctly — only where countries have genuinely different official names

---
### Per-Locale Generator
generate_locale_po_files.py reads holidays_l10n.json and generates one .po file per language containing every holiday in the library. Right now it produces 160 language files with 2452 entries each —for example bg.po contains Bulgarian translations for every holiday from Afghanistan to Zimbabwe, not just Bulgarian holidays.
- However, the generator has a strict requirement:every language entry in the JSON must be a flat string before it can run
- The reason for this strict requirement goes back to how Gettext works a .po file is a dictionary where one msgid maps to exactly one translation. There is no way to say "use this French translation in Belgium but this other one in Côte d'Ivoire" in the same file. If two countries genuinely use different names for the same holiday in the same language, they are by definition different holidays that need separate msgids, and that is exactly what the splitting work in Step 1 resolves before Step 2 can run cleanly.
- Once the JSON is fully verified and all genuine naming differences are correctly split into separate entries, the generator will produce clean output for all 160 languages. The number of entries will grow beyond 2452  each split adds one new entry. For example if "Independence Day" is currently one entry covering 93 countries but 15 of those have genuinely different official names, it becomes 16 entries after splitting. Each one gets its own msgid and its own translations in every language file.

---
### A New Translation Model — How It All Connects
The new translation model only works when all three pieces are in place together  and each one depends on the previous. This final step,  which changes how the library loads translations at runtime  and this is where everything comes together
- The JSON must be clean first. Every language entry must be a flat string no nested variants
- The generator must run cleanly before the country files can be updated. Once every nested variant is split into its own entry, the generator produces bg.po, fr.po, uk.po and 157 other language files  each containing every holiday in the library with that language's translation as msgstr. These files then get compiled to .mo binary files that Python's gettext module can read at runtime
The country files must be updated before the new model works end-to-end. Right now Bulgaria's country file still calls tr("Ден на Независимостта на България") using the Bulgarian string directly. But bg.mo no longer maps Bulgarian → Bulgarian. It maps English → Bulgarian. So tr("Independence Day of Bulgaria") → "Ден на Независимостта на България". replace_tr_strings.py does this replacement automatically across all 3557 call sites
Once all three are in place, the full system works like this:


<img width="1357" height="410" alt="image" src="https://github.com/user-attachments/assets/30166882-f3da-448c-a285-694b513bd880" />



- The same bg.mo file works for any country: if  Ukraine wants Bulgarian names for German holidays, holidays.Germany(language='bg') loads the same bg.mo and returns Bulgarian names for every German holiday. This is what the per-locale model makes possible one file per language, serving the entire library
----
### Pull Requests
| PR | Description | Status |
|---|---|---|
| [#3585](https://github.com/vacanza/holidays/pull/3585) | Generate l10n file on per locale basis (main GSoC PR) | Open |
| [#3596](https://github.com/vacanza/holidays/pull/3596) | Fix UAE Thai localization (from Linguistic work ) |  Merged |
| [#3678](https://github.com/vacanza/holidays/pull/3678) | Update Monaco holidays (from Linguistic work) |  Merged |

------
### Acknowledgments
- I am deeply grateful to my mentors for their time and guidance throughout this project. They reviewed work carefully, gave honest feedback, and pushed me to think through problems properly rather than just providing answers. That kind of engagement made this a genuinely valuable experience.
- Thank you to the Python Software Foundation for organizing and supporting GSoC, and to the Open World Holidays Framework (Vacanza) for welcoming me as a contributor and trusting me with a meaningful piece of their codebase. Working on a real library used by developers around the world was a privilege.
Thank you to Google for making this program possible. GSoC opened doors and created opportunities that will stay with me long after this summer  and I look forward to what comes next because of it

  

