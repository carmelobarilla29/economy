# CB29 Economy

App personale di finanze di Carmelo. Un solo file: `index.html`, servito da
GitHub Pages su https://carmelobarilla29.github.io/economy/ e installato sulla
home dell'iPhone come web app.

**Prima di modificare qualsiasi cosa, leggi tutto questo file.** Molte scelte
qui dentro sembrano semplificazioni o sviste: non lo sono, sono decisioni prese
insieme a Carmelo dopo averle discusse. Non "migliorarle" senza chiederglielo.

## Com'è fatta

Vanilla JS, niente build, niente dipendenze, niente framework. Tutto in
`index.html`: uno `<style>`, il markup minimo, e uno `<script>` con:

- **stato** — un oggetto `S` salvato in `localStorage` sotto `cb29economy.v1`
  (migra dalla vecchia chiave `soldicamera.v1`)
- **ledger** — un unico registro di movimenti; i saldi sono sempre calcolati
  sommando il registro, mai memorizzati
- **viste** — funzioni `vHome`, `vMov`, `vBiz`, `vPlay`, `vDebts`, `vLoans`,
  `vStats`, `vBackup`, ognuna ritorna una stringa HTML; `render()` la inietta
- **pannelli** — le funzioni `sheet*` aprono i moduli di inserimento

Font: Onest da Google Fonts. Grafici: SVG scritto a mano, nessuna libreria.

## Modello dei dati

```
S = {
  ledger: [{id, w, a, l, k, ref, d, c}],   // w=portafoglio a=importo l=etichetta
                                            // k=tipo ref=collegamento d=data c=categoria
  orders: [{id, name, date, wallet, total, ship2, ship2paid, ship2date,
            wallet2, share, arrived, arrivedDate, items:[{id,name,qty}]}],
  sales:  [{id, oid, iid, qty, price, channel, wallet, date}],
  play:   [{id, kind, amount, wallet, date, note}],
  debts:  [{id, name, total, left}],
  loans:  [{id, kind, person, total, left, wallet, date, note}],
  promos: [],   // sezione tolta, i dati vecchi restano qui dentro
  bk, pin, v
}
```

**Portafogli**: `contanti`, `conto`, `vinted`, `risparmi`. In più il valore
speciale `nessuno`, che significa "questo movimento è successo ma non tocca i
miei soldi" (usato per il gioco fatto prima di avere l'app e per gli ordini già
pagati altrove).

Se un giorno servisse un'altra piattaforma (Wallapop e simili) si aggiunge allo
stesso modo: `walletName`, le due `walletSel*`, `balanceAt`, e le tessere in
`vHome` e `vMov`. Sono cinque punti, non ce ne sono altri — ma **`balanceAt` è
quello che si dimentica**, e se lo dimentichi i grafici ignorano quel
portafoglio mentre i totali del mese no.

**`ref`** collega una riga del registro alla sua origine: `order:<id>`,
`order2:<id>`, `sale:<id>`, `play:<id>`, `giro:<uid>`, `rata:<debtId>:<importo>`,
`loan:<id>`, `rid:<loanId>:<importo>`.
Le righe con `ref` non si modificano dai Movimenti: si cambia l'oggetto
d'origine e le righe vengono riscritte. `unpost(ref)` cancella tutte le righe
di un ref.

## Le decisioni, e perché

**Il numero grande in home sono solo contanti + conto.** Non i risparmi, non il
magazzino, non il saldo Vinted, non i soldi che deve ancora riavere. È "quanto
posso spendere adesso". Sotto compare una riga con risparmi, saldo Vinted,
soldi da riavere e debiti, e il totale complessivo, che invece li conta tutti.

**Il saldo Vinted è un portafoglio come gli altri** (`daTrasferire()`). Quando
vende, i soldi entrano lì davvero: sono suoi e contano come entrata del mese,
ma per spenderli deve prima trasferirseli sul conto, e quel trasferimento è un
**giro** normale — quindi non risulta né entrata né spesa. Per questo sta fuori
dal numero grande: prima di quel giro, quei soldi in mano non ce li ha. La
tessera in home e la casella nei Movimenti compaiono solo se il saldo non è
zero, così chi non vende non se la ritrova fra i piedi.

**Il magazzino non è patrimonio.** I soldi di un ordine sono usciti e basta,
contano come spesa. In Business quel valore si chiama "Da rientrare": non è
roba che possiedi, è quanto devi ancora recuperare vendendo.

**I giri tra portafogli non sono né entrate né spese.** `monthInOut` e
`catBreakdown` li escludono. `balanceAt` somma tutti e tre i portafogli proprio
perché uno spostamento interno non deve risultare una perdita — è stato un bug
vero e va tenuto d'occhio.

**Il gioco sta fuori dai soldi.** Ha la sua tessera in home. Un deposito fatto
adesso esce dal conto; per quello che ha giocato prima dell'app si usa il
portafoglio `nessuno`. Il numero che conta è il saldo da sempre, non quello del
mese: serve che resti sempre visibile.

**Gli ordini si registrano col totale, mai col prezzo per pezzo.** Carmelo
inserisce quanto ha pagato in tutto e solo nome e quantità degli articoli. Il
costo per pezzo è il totale diviso i pezzi, uguale per tutti. È
volutamente approssimativo sul singolo modello ed esatto sul totale: chiedere i
prezzi unitari lo bloccava e non registrava più niente. **Non reintrodurre quel
campo.**

**Ordine e spedizione per l'Italia sono due pagamenti separati**, in due momenti
diversi, ognuno con il suo portafoglio e la sua data. Finché la spedizione non è
pagata l'ordine è "aperto" e il costo per pezzo è provvisorio. Poi passa
**in transito**, e solo quando Carmelo tocca il pulsante diventa **arrivato**:
pagare non vuol dire che il pacco è a casa.

**Ordini in società** (`share`, frazione da 0 a 1): la roba resta tutta a lui e
la vende lui, il socio ha una quota sui guadagni. Quindi:
- si inseriscono gli importi **pieni** dell'ordine
- dal portafoglio esce solo `totale × share`
- costo, "da rientrare", magazzino e profitto mostrati sono **i suoi**, cioè
  già moltiplicati per `share`
- `saleSocio()` calcola quanto di ogni incasso spetta al socio, e Business
  mostra "Da dare ai soci". Quando lo paga, registra una spesa normale.

**Soldi da riavere** (`S.loans`, `vLoans`, `sheetLoan`, `sheetRepay`): soldi
suoi che devono tornargli. Due tipi nello stesso elenco, distinti da `kind`:

- `kind:"prestito"` — li ha dati a una persona. Escono dal portafoglio (riga
  `prestito`, ref `loan:<id>`) ma **non sono una spesa**: `monthInOut` e
  `catBreakdown` li escludono come i giri.
- `kind:"arrivo"` — soldi suoi fermi altrove (un rimborso, un conto di gioco).
  Non sono mai stati nei portafogli, quindi **non scrivono niente nel ledger**:
  esiste solo la voce in `S.loans`.

Quando rientrano, in tutti e due i casi, una riga `rimborso` con ref
`rid:<loanId>:<importo>` porta i soldi nel portafoglio scelto. **Non è
un'entrata**: quei soldi erano già suoi, contati in `loanLeft()`.

`loanLeft()` entra nel totale complessivo in home ma **non nel numero grande**:
i soldi che devono ancora arrivare non si possono spendere adesso.

Non si può segnare un rientro più grande di quello che resta, come per le rate.
Eliminando una voce spariscono anche i rientri collegati. Le voci registrate
prima che esistesse il tipo `arrivo` non hanno `kind` e valgono come prestiti.

**La sezione Promo non c'è più.** È stata tolta perché Carmelo ha un'app a parte
per quelle. `S.promos` è rimasto nei dati e nel backup, invisibile: i suoi dati
vecchi non sono stati cancellati e la sezione si potrebbe rimettere. "Promo"
resta fra le categorie delle entrate.

**Le correzioni di saldo non sono soldi che si muovono.** Spostano il saldo (e
quindi i grafici, che sono saldi), ma restano fuori da ogni numero del tipo
"questo mese hai guadagnato X": la pillolina in home, l'intestazione del mese
nei Movimenti, "Differenza del mese" e "Dove vanno i soldi". Quelli usano
`monthReal()`, cioè entrate + uscite senza giri e senza rettifiche. La riga
"Correzioni di saldo" resta visibile a parte nel riepilogo del mese, così non
si nasconde niente. Serviva: alla prima installazione uno mette tutti i suoi
soldi con "Correggi saldo" e la home gli diceva "+650 € questo mese".

**Le date vuote** finiscono in un gruppo "Senza data", non fanno crashare
`labMonth`. **`num()`** interpreta il punto come separatore delle migliaia
(`1.234` = 1234), perché l'utente scrive all'italiana.

**Niente `confirm()` del browser**: usa `ask()`, un dialogo interno, perché il
confirm nativo può essere bloccato in certi contesti. `bind()` impedisce il
doppio tocco, e `showErr()`/`fail()` sbloccano subito dopo un errore di
validazione, altrimenti il secondo tocco veniva ignorato.

## Cose da non fare

- **Non aggiungere il prezzo per pezzo negli ordini.** Ci abbiamo rinunciato
  apposta.
- **Non far entrare magazzino, gioco o soldi da riavere nel numero grande.**
  Nel totale complessivo sotto, invece, i soldi da riavere ci vanno.
- **Non trasformare i giri in entrate/uscite.**
- **Non introdurre un blocco che sovrascrive `S` all'avvio.** Ne è esistito uno
  (caricava i dati iniziali) ed è stato rimosso apposta: era una mina, bastava
  toccarne l'identificativo per cancellare tutto quello che Carmelo aveva
  registrato.
- **Non aggiungere dipendenze, build step o framework.** Un file solo, si carica
  su GitHub e va.
- **Non usare emoji** nell'interfaccia: le icone sono SVG disegnate a mano.

## Prima di consegnare una modifica

I dati vivono solo in `localStorage`, su un solo telefono. Un errore qui non è
un bug estetico: sono i suoi soldi. Prima di dirgli che è fatto:

1. Aprire il file in un browser e provare davvero il flusso toccato
2. Controllare che i saldi non siano cambiati se non dovevano
3. Se cambi qualcosa nei calcoli, verificarlo con numeri veri

Nel repo non c'è una suite di test. Se ne aggiungi una, Playwright su Chromium
funziona bene: gli elementi hanno attributi `data-*` stabili (`data-go`,
`data-sheet`, `data-led`, `data-order`, `data-adj`…) pensati proprio per essere
agganciati.

## L'app gemella di Alessandro

`~/Documents/economy-alessandro` → https://github.com/carmelobarilla29/economy-alessandro,
pubblicata su https://carmelobarilla29.github.io/economy-alessandro/. Stesso
codice, progetto separato, gestito sempre da Carmelo: una modifica a un'app non
tocca l'altra.

Le due app stanno sullo stesso sito e il browser lega i dati al sito, non alla
cartella: a tenerle separate è solo la chiave in cima allo script
(`cb29economy.v1` qui, `aleeconomy.v1` di là). **Non vanno mai rese uguali né
cambiate**: cambiarne una cancella i dati di chi la usa.

Differenze: di là c'è ancora la sezione Promo e non c'è il recupero da
`soldicamera.v1`. Il resto, "Soldi da riavere" compreso, è identico.
**Non copiare `index.html` da un progetto all'altro**: ti porteresti dietro
chiave, nome e saluto sbagliati. Le modifiche si riportano a mano.

## Aggiornare l'app

`index.html` sta nella root del repo. GitHub Pages pubblica da `main`. Un
commit e un push, un minuto, e l'app sul telefono è aggiornata: non c'è deploy
da fare. I dati di Carmelo non vengono toccati da un aggiornamento del codice.

## Come parlargli

Carmelo ha 18 anni, è studente, non è un programmatore. Guadagna con le promo
referral e rivendendo prodotti presi da fornitori cinesi. Spiegagli le cose in
italiano e in parole normali: "il numero in alto", non "il campo `liquid()`".
Quando una scelta ha un effetto collaterale sui suoi numeri, diglielo prima,
non dopo.
