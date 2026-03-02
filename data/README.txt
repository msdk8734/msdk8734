Population JSON files generated from major_results_2020.xlsx (2020 Census major results).
- pref-pop.json: prefecture-level population data
- municipalities/muni-pop-XX.json: municipality-level population data per prefecture
Fields:
  code, prefCode (municipalities only), nameJa, nameEn, total, male, female

English display names are normalized for natural UI display, e.g.:
- Wajima City
- Chuo Ward, Sapporo City
- Ishikawa Prefecture

Note:
- 07546 Futaba Town is missing total/male/female in the source workbook, so values are null.
