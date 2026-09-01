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
- **viste** — funzioni `vHome`, `vMov`, `vBiz`, `vPromo`, `vPlay`, `vDebts`,
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
  promos: [{id, person, type, date, value, tokens, status, into}],
  play:   [{id, kind, amount, wallet, date, note}],
  debts:  [{id, name, total, left}],
  bk, pin, v
}
```

**Portafogli**: `contanti`, `conto`, `risparmi`. In più il valore speciale
`nessuno`, che significa "questo movimento è successo ma non tocca i miei
soldi" (usato per il gioco fatto prima di avere l'app e per gli ordini già
pagati altrove).

**`ref`** collega una riga del registro alla sua origine: `order:<id>`,
`order2:<id>`, `sale:<id>`, `play:<id>`, `giro:<uid>`, `rata:<debtId>:<importo>`.
Le righe con `ref` non si modificano dai Movimenti: si cambia l'oggetto
d'origine e le righe vengono riscritte. `unpost(ref)` cancella tutte le righe
di un ref.

## Le decisioni, e perché

**Il numero grande in home sono solo contanti + conto.** Non i risparmi, non il
magazzino, non i buoni promo. È "quanto posso spendere adesso". Sotto compare
una riga con risparmi e debiti e il totale complessivo.

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

**Promo**: 7600 isytoken = 30 € (`TOKEN_EUR`). Stati: invitato → registrato →
accreditato → convertito → speso. Accreditato e convertito contano come "buoni
da usare"; escono dal totale solo con "speso" — un buono Amazon convertito ce
l'ha ancora in mano. Il campo "convertito in" è testo libero.

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
- **Non far entrare magazzino, buoni promo o gioco nel numero grande.**
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
