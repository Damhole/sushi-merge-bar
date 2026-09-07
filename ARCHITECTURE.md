# Sushi Merge Bar — Architektura

Tenhle dokument popisuje herní logiku, datový model a to, jak jsou části kódu napojené. Programátor by měl po přečtení vědět, kde co je a kde co změnit.

**Celá hra je jeden soubor:** [`index.html`](index.html) (~1058 řádků, vanilla HTML + CSS + JS, žádné závislosti, žádný build). Otevře se dvojklikem, nebo přes libovolný statický server.

Live: https://damhole.github.io/sushi-merge-bar/

---

## 1. Přehled

Overcooked-style merge puzzle. Hráč sbírá suroviny z mřížky, dává je na pult, a jakmile jich tam má 3, co tvoří recept některé viditelné objednávky, automaticky se smerguje.

Herní cyklus (5 fází):

1. **Start levelu** — načte se JSON levelu, mřížka je plná surovin, fronta objednávek se vygeneruje ze seeded shuffle
2. **Sbírání** — hráč kliká na aktivní suroviny (musí mít volné sousední pole). Klik pošle surovinu do pultu (7 slotů).
3. **Auto-merge** — po každém tahu se prohledá, jestli 3 suroviny v pultu tvoří multiset některé viditelné objednávky. Když ano → mizí, objednávka splněna.
4. **Výhra** — mřížka prázdná + fronta receptů prázdná = win overlay.
5. **Game over** — pult 7/7 + žádná viditelná objednávka se nedá splnit + suroviny v mřížce = orange overlay.

---

## 2. Layout (HTML struktura)

```
<body> — flex center, min-height 100vh, padding 16px
├── <div class="app">                         # telefonový rámec (max 360×932)
│   ├── <header>                              # stepper + reset button
│   │   └── <div class="header-right">
│   │       ├── <div class="level-stepper">   # [− N +] přepínač levelů
│   │       └── <button class="btn-icon">↻    # reset aktuálního levelu
│   ├── <div class="orders-wrap">             # lístečky na kuchyňské liště
│   │   ├── <div id="task-title">             # velké centrové skóre (jen timed level)
│   │   └── <div id="recipes">                # horizontální řada order-card
│   ├── <div class="grid-area">               # pevný čtverec 360×360
│   │   └── <div id="grid">                   # grid cols×rows, cells sized JS-em
│   └── <div class="panel stash-wrap">        # pult / zásobník 7 slotů
│       └── <div id="stash">
├── <div class="overlay" id="overlay-win">    # Level splněn
├── <div class="overlay" id="overlay-over">   # Game Over
└── <div class="toast" id="toast">            # dočasné hlášky
```

**Klíčové rozhodnutí layoutu:**
- `.app` uchycený k vrchní hraně (`body { align-items: flex-start }`)
- `.grid-area` má `aspect-ratio: 1` = pevný čtverec, mřížka se v něm bottom-aligned
- Pult (`stash-wrap`) je pak vždy na konzistentním Y (nezávislé na cols/rows levelu)

---

## 3. Datový model

### 3.1 Suroviny (`TYPES`)

Statické mapování `type → vizuál`. Nejsou to instance, jen šablony.

```js
const TYPES = {
  salmon:   { label: "L", icon: "🐟", cls: "t-salmon"   },
  rice:     { label: "R", icon: "🍚", cls: "t-rice"     },
  seaweed:  { label: "Ř", icon: "🌿", cls: "t-seaweed"  },
  tuna:     { label: "T", icon: "🐠", cls: "t-tuna"     },
  avocado:  { label: "A", icon: "🥑", cls: "t-avocado"  },
  ginger:   { label: "Z", icon: "🫚", cls: "t-ginger"   },
  cucumber: { label: "C", icon: "🥒", cls: "t-cucumber" },
  crab:     { label: "K", icon: "🦀", cls: "t-crab"     },
};
```

- `label` — jednopísmenný kód, používá se ve stringové mapě levelu (viz níže)
- `icon` — emoji renderované v dlaždici
- `cls` — CSS class pro barvu (`.t-salmon { background: #FF6B6B }` atd.)

**Přidat surovinu:** doplň entry sem, přidej CSS `.t-<name>`, dopiš do `CHAR_TO_TYPE`.

### 3.2 Recepty

Recept = `{ name, items[3] }`. `items` je pole 3 typů (multiset — pořadí nezáleží).

```js
const MAKI    = { name: "Maki",    items: ["salmon", "rice", "seaweed"] };
const NIGIRI  = { name: "Nigiri",  items: ["tuna",   "avocado", "ginger"] };
const URAMAKI = { name: "Uramaki", items: ["rice",   "cucumber", "crab"] };
```

Recepty **můžou sdílet suroviny** (Maki i Uramaki potřebují rýži). Auto-merge to řeší podle multisetu (viz `findMatch`).

### 3.3 Instance objednávky

V herním stavu (`recipes[]`) je každá objednávka:

```js
{
  name: "Maki",             // display name
  items: ["salmon","rice","seaweed"],  // multiset požadovaných typů (kopie)
  done: false,              // true po splnění
  spawnedAt: 1234567890     // ms timestamp — kdy vstoupila do viditelné fronty (jen timed level)
}
```

### 3.4 Mřížka

`grid` = `Map<"col,row", { type, col, row, hidden? }>`. Klíč je string `"c,r"` přes helper `key(c,r)`.

```js
grid.set("2,3", { type: "salmon", col: 2, row: 3, hidden: false });
```

- Pole bez surovinu = klíč není v mapě (ne `undefined`, prostě chybí)
- `hidden: true` = mystery box, zobrazí `?` dokud není políčko přístupné
- `cols` a `rows` jsou globální podle aktuálního levelu

### 3.5 Pult (`stash`)

Pole až 7 instancí. Každá instance má `origin` (odkud přišla) — historicky sloužilo pro vrácení surovin do mřížky (mechanika později odstraněna, ale `origin` zůstal pro budoucí použití).

```js
stash = [
  { type: "salmon", origin: { col: 0, row: 3 } },
  { type: "rice",   origin: { col: 1, row: 3 } },
];
```

### 3.6 Definice levelu

```js
{
  name: "Level 4 — Combo Bar",
  timed: false,             // true = časový tlak + skóre (jen L6)
  map: [                    // stringová mapa: velké písmeno = viditelná surovina,
    "ŘŘŘZZZ",               //                 malé písmeno = mystery box (?),
    "Řřtzzt",               //                 tečka nebo mezera = prázdné pole
    "AATtLL",
    "AAtRlL",
    "ARRRLR",
  ],
  recipes: shuffledOrders([  // pole objednávek – lze použít helper shuffledOrders
    { recipe: MAKI,    count: 4 },
    { recipe: NIGIRI,  count: 4 },
  ], 7),                     // seed pro deterministický shuffle
}
```

Alternativně místo `map` může level definovat `cols`, `rows` a `ingredients: [{ type, col, row }]` (starší formát, používá L1, L2).

**`CHAR_TO_TYPE`** mapuje písmena v mapě na typy. Pro nové suroviny sem přidej entry.

---

## 4. Herní stav (globální proměnné)

```js
let levelIndex = 0;   // aktuální level (0-based)
let grid;             // Map "c,r" → ingredient
let stash;            // pole [0..7] instancí
let recipes;          // pole objednávek pro aktuální level
let cols, rows;       // rozměry mřížky
let ordersSig = null; // signature fronty – karty se překreslí jen při změně
let score = 0;        // skóre (jen timed level)
let rafId = null;     // requestAnimationFrame id pro timer loop
```

Všechno na globální scope. Není to elegantní, ale pro prototyp v jednom souboru je to čitelné.

---

## 5. Herní logika (funkce → co dělá)

### 5.1 Načtení levelu

**`loadLevel(idx)`** (řádek 696) — reset všeho + parsing definice levelu:
1. Vyresetuje `grid`, `stash`, `recipes`, `score`, `ordersSig`.
2. Podle `lvl.map` nebo `lvl.ingredients` naplní `grid` a nastaví `cols`, `rows`.
   - Písmena v mapě jdou přes `.toUpperCase()` — malé písmeno detekováno porovnáním, nastaví `hidden: true`.
3. Vytvoří kopie receptů (aby `done` bylo per-instance).
4. Zavolá `renderLevelStepper()`, skryje overlays, `render()`.

### 5.2 Přístup k surovinám

**`isActive(c, r)`** (řádek 747) — může hráč tuhle surovinu vzít?

Pravidlo: surovina má aspoň jedno volné sousední pole (nahoře/dole/vlevo/vpravo v rámci mřížky), nebo je ve spodní řadě (spodní okraj = výdej).
Boční a horní okraj mřížky NEjsou "volné" — musí být skutečné prázdné políčko uvnitř.

```js
function isActive(c, r) {
  if (r === rows - 1) return true;  // spodní řada = vždy aktivní
  const dirs = [[0,-1],[0,1],[-1,0],[1,0]];
  return dirs.some(([dc, dr]) => {
    const nc = c + dc, nr = r + dr;
    if (nc < 0 || nr < 0 || nc >= cols || nr >= rows) return false;
    return !grid.has(key(nc, nr));
  });
}
```

### 5.3 Odeslání suroviny na pult

**`sendToStash(c, r)`** (řádek 758):
1. Zkontroluje, že políčko má surovinu.
2. Zkontroluje `isActive` a `stash.length < STASH_SIZE`.
3. Odstraní z gridu, přidá do `stash` s `origin`.
4. `render()`.
5. Po 120ms zavolá `tryMerge()` (nechá se stihnout render/animace).

### 5.4 Hledání shody

**`findMatch(recipe)`** (řádek 774) — vrátí pole 3 indexů v pultu, které tvoří multiset receptu, nebo `null`.

Prochází pult greedy: pro každý item se snaží najít odpovídající typ, dokud nemá plnou sadu.

### 5.5 Merge

**`tryMerge()`** (řádek 790) — main auto-merge loop:

```
while (guard < 20):
  for i in visibleRecipeIndices():
    match = findMatch(recipes[i])
    if match:
      animateMerge(match)
      odstranit ty 3 indexy z pultu (od konce)
      score += getOrderPoints(i)   // 100/60/20 podle timeru (jen timed)
      flyPoints(...)               // +N zlatá animace nad kartou
      recipes[i].done = true
      break
  else:
    break  // žádná shoda, konec

if merged:
  render(); flashStash("ok"); checkWin() && return
checkGameOver()
```

**Důležité:** hledá se jen ve **viditelných** objednávkách (prvních 4 nedokončených). Když má hráč v pultu suroviny pro 5. objednávku ve frontě, nesplní se, dokud se objednávka neposune do viditelné čtyřky.

### 5.6 Kontrola konce

**`checkWin()`** — grid prázdný + všechny recepty `done`? Ukázat win overlay (s skóre X/Y na timed levelu).

**`checkGameOver()`** — pult plný (7/7) + žádná viditelná objednávka se nedá splnit + v mřížce zbývají suroviny? Ukázat game-over overlay.

Vracení surovin z pultu do mřížky **není** povolené (design decision — přidává tenzi).

---

## 6. Rendering

### 6.1 `render()` — dispatch

```js
function render() {
  renderRecipes();  // lístečky objednávek
  renderGrid();     // mřížka
  renderStash();    // pult
}
```

Volá se po každé změně stavu. Ne-timed dopady jsou minimální — celý DOM se přerenderuje.

### 6.2 `renderGrid()` (řádek 938)

Klíč přizpůsobivosti — počítá velikost dlaždice tak, aby čtverec `cols×rows` fitl do fixní čtvercové `.grid-area`:

```js
const areaW = area.clientWidth  || 360;
const areaH = area.clientHeight || 360;
const cellSize = Math.floor(Math.min(
  (areaW - gap * (cols - 1)) / cols,
  (areaH - gap * (rows - 1)) / rows
));
g.style.gridTemplateColumns = `repeat(${cols}, ${cellSize}px)`;
g.style.gridAutoRows        = `${cellSize}px`;
```

Pak pro každé políčko:
- Když má surovinu → `makeIngredient(type, onClick, active, hidden, justRevealed)`.
- Když je hidden a stala se aktivní → nastaví `hidden = false` (reveal) a přidá CSS class `.reveal` na animaci.

**Resize handler** (řádek 1051) je debounced přes `requestAnimationFrame` a volá `renderGrid()`.

### 6.3 `renderRecipes()` (řádek 867)

Používá `ordersSig` cache — DOM se přepisuje jen při skutečné změně fronty (`done/total | visibleIndices | timed`). Bez toho by se lístečky překreslovaly při každém kliku a nešel by udržet timer bar animace.

**Timed karta** (L6):
```html
<div class="order-card timed" data-recipe-idx="0">
  <div class="card-head">
    <div class="smiley"><span class="mouth"></span></div>
    <div class="timebar"><div class="timebar-fill"></div></div>
  </div>
  <div class="card-body">
    <span class="card-name">Nigiri</span>
    <span class="card-ings">🐠🥑🫚</span>
  </div>
</div>
```

**Ne-timed karta** (L1-L5) — jen `card-name` + `card-ings`, bez head.

Když objednávka vstoupí do viditelné fronty a nemá `spawnedAt`, nastaví se `Date.now()`.

### 6.4 `renderStash()`

Jednoduché — 7 slotů, každý buď prázdný nebo s `makeIngredient(type, () => {})` (klik nedělá nic, kurzor `default`).

### 6.5 `makeIngredient()`

Vyrobí DOM element pro surovinu. Podle flagů:
- `hidden` → mystery box s `?`
- `!active` → přidá class `locked` (šedá)
- `justRevealed` → přidá class `reveal` (pop animace)

---

## 7. Timer / scoring (L6 „Rush Hour")

Aktivní jen na `LEVELS[i].timed === true`.

### 7.1 Konstanty

```js
const ORDER_DURATION = 30000;     // 30s per objednávka
const PHASE_GREEN_END  = 0.40;    // 0–40% = 100 bodů
const PHASE_YELLOW_END = 0.75;    // 40–75% = 60 bodů, pak 20
```

### 7.2 `phaseFromElapsed(elapsed)` (řádek 633)

Vrátí objekt s barvou baru, výrazem smajlíka, body a % pro width:
```js
{ phase: "green"|"yellow"|"red", pts: 100|60|20,
  color: "#43A047"|"#FB8C00"|"#E53935",
  smileyCls: ""|"neutral"|"sad",
  fillPct: 0..100 }
```

### 7.3 `tickOrders()` (řádek 656)

Volá se z `requestAnimationFrame` loopu (`startTimerLoop`). Prochází všechny viditelné `.order-card.timed[data-recipe-idx]`, spočítá elapsed a přepíše `.timebar-fill` width/background + `.smiley` class + background. Bez full re-renderu.

### 7.4 Skóre

Při mergi: `score += getOrderPoints(idx)`. `getOrderPoints` vrátí body podle aktuální fáze (nebo 100 na ne-timed).

`flyPoints(idx, pts)` = zlatá `+100` animace, která vyletí nad kartou.

Skóre v hlavičce panelu je `<span class="score-display">` — updatuje se ze `renderRecipes()`.

---

## 8. Levely (aktuálně 6)

| # | Grid | Typy surovin | Objednávky | Speciality |
|---|---|---|---|---|
| 1 | 4×4, 3 suroviny | 3 (Maki) | 1× Maki | Základní tutorial |
| 2 | 4×4, 6 surovin | 3 (Maki) | 2× Maki | Víc objednávek |
| 3 | 6×4 plná | 3 (Maki) | 8× Maki | Blocker mechanika (přístup ze všech stran mimo horní/boční okraj) |
| 4 | 6×5 | 6 (+Nigiri) | 10× mix, seed 7 | Mystery boxy (malá písmena), 2 recepty |
| 5 | 6×6 | 8 (+Uramaki) | 12× mix, seed 13 | Uramaki sdílí rýži s Maki |
| 6 | 6×5 | 8 | 10× mix, seed 21 | **Timed** — smajlík + bar + skóre |

Definice jsou v `const LEVELS = [...]` (řádek 527). Přidání levelu = přidat entry a je hotovo.

---

## 9. UI mechaniky

### 9.1 Objednávkové lístky (Overcooked styl)

- **Papírový vzhled**: `#fffaf0` bg, box-shadow, border-radius 6.
- **Klip nahoře**: `.order-card::before` = tmavý obdélníček zavěšený na liště.
- **Lišta**: `.recipes::before` = tenká vodorovná linka za lístečky.
- **Aktivní karta**: zlatý ring (první nedokončená v pořadí).
- **Zobrazuje se jen 4** (`VISIBLE_ORDERS`). Zbytek je „v kuchyni" — hráč nevidí co ho čeká.
- **Auto-shift**: když se karta dokončí, další doplní zprava (animace `cardIn`).

### 9.2 Přepínač levelů

`[− N +]` kapsle + `↻` reset, vpravo nahoře. Prev/Next disabled na krajích. Reset se otočí o 360° při klepnutí (CSS transition).

### 9.3 Overlays

- **Win**: zelený background, „Level splněn!", tlačítko „Další level" (nebo „Hrát znovu" na L6).
- **Game Over**: oranžový background, tlačítko „Reset".

### 9.4 Toast

Dole plovoucí bublinka pro chybové hlášky (`toast("Zablokováno – ...")`).

---

## 10. Rozšiřování / kde co změnit

| Chci... | Kam jít |
|---|---|
| Přidat novou surovinu | `TYPES` + CSS `.t-<name>` + `CHAR_TO_TYPE` |
| Přidat nový recept | Definuj `const NEWRECIPE = {...}`, přidej do `shuffledOrders` v levelu |
| Přidat nový level | Nový entry v `LEVELS`, definuj `map` nebo `ingredients`, `recipes` |
| Změnit počet slotů v pultu | `STASH_SIZE` (řádek 468) |
| Změnit počet viditelných objednávek | `VISIBLE_ORDERS` |
| Změnit trvání objednávky | `ORDER_DURATION` |
| Změnit hranice fází / body | `PHASE_GREEN_END`, `PHASE_YELLOW_END`, hodnoty v `phaseFromElapsed` |
| Změnit pravidlo přístupu k surovině | `isActive(c, r)` |
| Změnit vzhled lístečku | CSS `.order-card` a `.order-card.timed` |
| Přidat blocker/překážku | Rozšíř `grid` entry o `blocker: true`, uprav `isActive` a `renderGrid` |
| Umožnit vracení surovin | Odkomentuj/reimplementuj `returnToGrid`, přidej click handler v `renderStash` |

---

## 11. Testovatelnost

Solver, kterým jsem ověřoval dohratelnost levelů, běží přímo v konzoli prohlížeče. Jeho podobu najdeš v git historii (v commit messages), ale zjednodušená verze:

```js
async function solve() {
  loadLevel(levelIndex);
  while (grid.size > 0) {
    const act = [];
    grid.forEach(v => { if (isActive(v.col, v.row)) act.push(v); });
    if (!act.length) break;
    const stashTypes = new Set(stash.map(s => s.type));
    const pick = act.find(v => !stashTypes.has(v.type)) || act[0];
    sendToStash(pick.col, pick.row);
    await new Promise(r => setTimeout(r, 50));
  }
  return { done: recipes.filter(r => r.done).length, score };
}
```

Vlepený do DevTools konzole naschválu prochází levelem s greedy strategií. Užitečné pro ověření, že nový level je dohratelný (grid se vyklidí a nezůstanou přebytky).

---

## 12. Nedořešené / dobré nápady na později

- **Star rating na win** (★★★ za >80% skóre atd.) — code hint v conversation history
- **Penalizace za expired objednávku** (host odejde, mínus body)
- **Zvuk** (merge sound, kritický timer sound)
- **Persistent best score** přes localStorage
- **Level editor** — modal, kde nakreslíš mřížku, vygeneruje se JSON

---

Any questions, look at git log — commit messages jsou dost popisné a mapují dobře na jednotlivé milníky vývoje.
