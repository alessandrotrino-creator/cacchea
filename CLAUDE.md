# Cacchea

Gioco di dadi da poker per la famiglia, in **un solo file** (`index.html`): partita libera tra giocatori al tavolo e modalità storia contro avversari automatici. Interfaccia e testi in **italiano**. Niente build e niente dipendenze, a parte i caratteri da Google Fonts (Bowlby One e Nunito, con caratteri di riserva). HTML, CSS e JS vanilla sono tutti nella pagina. Lo stile riprende la repo `alessandrotrino-creator/yahtzee`, che è dello stesso utente.

## File

- `index.html`: tutto il gioco. È l'unico file da pubblicare (per esempio su GitHub Pages).
- `README.md`: descrizione per GitHub.

Su questo PC non ci sono git, Python e Node. Per provare la pagina in automatico si può usare Edge headless: `msedge --headless=new --screenshot=out.png --window-size=480,1500 file:///...`. Per le simulazioni si usa `--dump-dom --virtual-time-budget=200000`.

## Regole del gioco (decise dall'utente)

- 5 dadi da poker. Facce: 0=9, 1=10, 2=J, 3=Q, 4=K, 5=Asso.
- Punti: Asso singolo 100, Q singola 50. Tris: Asso 700, K 600, Q 500, J 400, 10 300, 9 200.
- Il tris vale solo se i tre dadi escono **nello stesso lancio**.
- Si bloccano **solo dadi che fanno punti**: almeno uno, al massimo tutti quelli che valgono. Si può bloccarne solo una parte, per esempio una Q su due. Due K da soli non si possono bloccare. Bloccare è l'unico modo di tenere i dadi.
- Se il lancio non contiene un Asso, una Q o un tris è **CACCHEA**: si perdono i punti del turno.
- Se si bloccano tutti e 5 i dadi, si rilanciano tutti e 5 e i punti restano.
- Ci si può fermare quando si vuole e segnare i punti.
- **Apertura**: la prima volta in ogni partita servono almeno 600 punti in un turno (`OPEN`).
- **Chiamarsi fuori**: oltre il punteggio scelto (5000 di default, `OUT`; in partita è `G.target`), prima di tirare. Gli altri fanno un ultimo turno a testa, poi vince chi ha più punti. Chi si chiama fuori non tira più.
- **Poker e cinque uguali** (confermato dall'utente): contano come tris più gli Assi o le Q in più.
  - 4 Assi = 800.
  - 5 K: si blocca il tris (600) e si rilanciano gli altri due. I dadi in più di K, J, 10 e 9 non si possono bloccare.
  - 4 Q: si bloccano tutte (550) e si tira un dado, oppure si tiene una Q sola (50) e si tirano 4 dadi.

## Grafica

- Dadi in SVG (`dieSVG`), **esattamente come carte da poker**: i dadi veri dell'utente sono fatti così. L'indice è in alto a sinistra e capovolto in basso a destra.
  - Asso: picche nera piccola al centro.
  - K rosso, Q verde scuro, J blu: figura a due teste dentro una cornice, con linea diagonale in mezzo (`halfFigure()` + `figure()`).
  - 10: dieci picche rosso spento (4 + 4 laterali e 2 centrali).
  - 9: nove picche blu scuro (4 + 4 laterali e 1 centrale).
  - Nel 9 e nel 10 le picche della metà bassa sono capovolte, come sulle carte. Le posizioni sono in `PIPS`.
- Cacca in SVG (`poopSVG(lv)`): a ogni cacchea si vede una cacca più grande in base ai punti persi (`POOP`):

  | Punti persi | Scritta | Effetti in più |
  |---|---|---|
  | meno di 600 | Cacchea! | |
  | 600 | Cacchea grossa! | puzza |
  | 800 | Caccona! | |
  | 1000 | Caccone! | mosche e schermo che trema |
  | 1500 | CACCONE GIGANTE! | |

- Suoni sintetizzati con WebAudio: dadi, pernacchia (`sndFart`, più lunga se la cacchea è più grossa) e toni.
- Tema unico: tavolo di panno verde con bordo di legno. Non ha la variante chiara.

## Struttura dello script (in ordine)

1. **LOGICA** (tra i commenti `LOGICA (inizio)` e `LOGICA (fine)`): `evalSel()`, `scores()`, `bestSel()`, `options()`, avversari automatici (`aiChoose()`, `aiCallOut()`, `aiReroll()`, `naiveStop()`).
   - Questo blocco non deve usare il DOM né `S`: viene estratto così com'è per le simulazioni.
2. **Storia**: `LEVELS` (5 tavoli da 5 avversari), `MINIS`, `PATH` (livello, minibonus, livello…), `POWERS` (aiuti), `TIRO_GOALS`, `levelReward()`.
3. **Dati salvati**: `S` in `localStorage['cacchea-v1']`, `defaults()`, `story(pid)`.
4. **Disegni e suoni**.
5. **Viste**: `render()` sceglie tra `viewSetup()` (partita libera), `viewStory()`, `viewGame()`, `viewEnd()`, `viewTiro()` (minibonus Tiro a segno).
6. **Partita**: `makeGame()`, `startTurn()`, `roll(commit)`, `bank()`, `cacchea()`, `savePoop()`, `nextTurn()`, `callOut()`, `finishGame()`, `finishMano()`.
7. **Avversari**: `aiGo()` / `aiAct()` giocano un passo alla volta con `setTimeout`. Si possono velocizzare con `S.aiFast`.
8. **Eventi**: un unico listener `click` su `document` che smista su `data-act`. Per un nuovo pulsante usa `data-act="nome"` e aggiungi un `case`.
   - Non si usano `alert`/`confirm`: la conferma è `ask()`, una finestra interna.

## Modello dati

- `S.roster`: giocatori `{id, name, emoji, preset?}`. Quelli precaricati sono gli stessi dello Yahtzee: Ale, Laura, Roby, Jo, Sere, Fra, Steu, Monica.
- `S.pick`: ordine dei giocatori della partita libera.
- `S.target`: punteggio per chiamarsi fuori (2000, 3000, 5000, 8000 o 10000).
- `S.G`: partita in corso, oppure `null`.
  - Campi: `{players, cur, final:{by}|null, over, log, turn, bigPoop, target, story?, mini?, fixMode?}`.
  - `turn` è `{pts, held, roll, sel, naive?}`.
  - I giocatori automatici hanno `ai:{skill, greed, luck}`.
  - `story` è `{pid, step, lv, n}`. `mini` è `{pid, step, left}`: è il minibonus "Una mano sola", che riusa la partita normale con un solo giocatore, già aperto, per 3 turni.
- `S.tiro`: minibonus Tiro a segno in corso.
- `S.story[pid]`: `{step, coins, pw:{roll, fix, clover}, nOpp, wins, done}`.
- Altri campi: `S.hero`, `S.tab`, `S.sound`, `S.aiFast`.

## Modalità storia: taratura

Gli avversari hanno tre parametri:

- `skill`: probabilità di giocare "da bravo". Un avversario bravo tiene pochi dadi per rilanciarne di più (`KEEP_VALUE`) e si ferma secondo `STOP_AT`. Uno "a istinto" tiene tutto e si ferma a una soglia casuale.
- `greed`: moltiplica le soglie di fermata.
- `luck`: se è positiva, a volte ritira una cacchea. Se è negativa, a volte ritira un lancio buono.

Simulazione (400 partite per riga, fino a 5000 punti), contro un giocatore umano di riferimento (`skill:.8, greed:1`). È la percentuale di vittorie dell'umano:

| Livello | 1 avversario | 2 avversari | 3 avversari |
|---|---|---|---|
| 1 Osteria | 67% | 56% | 48% |
| 2 Bar | 60% | 49% | 43% |
| 3 Sagra | 54% | 49% | 33% |
| 4 Casinò | 44% | 28% | 22% |
| 5 Re Caccone | 35% | 19% | 14% |

Una partita a 5000 dura circa 16-21 giri; a 2000 circa 7-9 giri. Il gioco è molto fortunoso, quindi le differenze tra i livelli si ottengono soprattutto con `luck`. **Se cambi un parametro, risimula**: estrai il blocco LOGICA più `LEVELS` in una pagina di prova ed eseguila con Edge headless.

## Preferenze dell'utente

- Testi e risposte in italiano, tono semplice: gioca tutta la famiglia.
- Le regole del gioco le decide l'utente: in caso di dubbio chiedi, non inventare.
- Emoji: Windows 10 non mostra quelle di Unicode 13 e successive. Per esempio usa 💰 e non 🪙.
