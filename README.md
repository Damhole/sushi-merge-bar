# Sushi Merge Bar

Hratelný prototyp 2D merge hry — vybíráš suroviny z mřížky, skládáš je na pult a recepty se automaticky merguje.

**[Hraj online →](https://damhole.github.io/sushi-merge-bar/)**

## Mechaniky

- **Click-to-send** — klikneš na surovinu, skočí na první volný slot pultu (7 slotů).
- **Auto-merge** — jakmile pult obsahuje suroviny tvořící některý recept, samy zmizí.
- **Žádné vracení** — co dáš na pult, tam zůstane do mergeu. Plánuj.
- **Přístup z volné strany** — surovina je sebratelná, jen když má vedle sebe aspoň jedno volné pole nebo je ve spodní řadě.
- **Mystery boxy** — některé suroviny jsou skryté pod `?`, odhalí se až se odemknou.
- **Viditelné jen 4 objednávky** — co je „v kuchyni" nevíš.

## Levely

| # | Co se učíš |
|---|---|
| 1 | Základní cyklus: 1 recept, 3 suroviny |
| 2 | Více objednávek za sebou |
| 3 | Blocker mechanika — kopat se odshora dolů |
| 4 | Combo Bar — 2 pokrmy (Maki + Nigiri), 6 typů surovin, 10 objednávek v náhodném pořadí, mystery boxy |
| 5 | Triple Threat — 3 pokrmy (přibyl Uramaki = rýže + okurka + krab, sdílí rýži s Maki), 8 typů surovin, 6×6 mřížka, 12 objednávek |
| 6 | Rush Hour — časový tlak. Každá viditelná objednávka má 30s: zelená 0–12s = 100b, oranžová 12–22.5s = 60b, červená 22.5s+ = 20b. Smajlík + bar nad kartou. Celkové skóre. |

## Spustit lokálně

Bez závislostí, žádný build. Stačí otevřít `index.html` v prohlížeči.
