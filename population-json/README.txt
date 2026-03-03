Population JSON bundle based on:
Ministry of Internal Affairs and Communications, Basic Resident Registration.
2025 "Total" municipal population, households, and 2024 vital events.

Source file:
2025【総計】令和7年住民基本台帳人口・世帯数、令和6年人口動態（市区町村別）EXCEL.xlsx

Files:
- pref-pop.json
- municipalities/muni-pop-01.json ... municipalities/muni-pop-47.json

Record schema:
- code
- prefCode (municipality files only)
- nameJa
- nameEn
- total
- male
- female

Notes:
- Prefecture totals come from prefecture total rows in the source workbook.
- Municipality files exclude aggregate rows such as county-only rows ending with "郡".
- Tokyo aggregate row "島しょ" is excluded.
- Rows marked "（再編前）" are excluded.
- nameEn values are carried forward from the previous JSON set so the display naming style stays consistent.
