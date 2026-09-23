# 02 — Context compaction

Costruisce un function hook che sostituisce la compattazione nativa di Claude Code: invece di riassumere la conversazione, chiede a Jev quali tool call servono ancora ed elimina o tronca le altre. Tutto ciò che resta nel contesto è verbatim. La compattazione parte alla percentuale di contesto che scegli durante il setup.

**Prerequisiti:** Claude Code ≥ 2.1.276, una API key TypeSafe (<https://console.typesafe.ai/keys>) in `TYPESAFE_API_KEY`.

**Uso:** apri `claude` nella cartella del progetto, incolla il prompt qui sotto e rispondi alla domanda sulla soglia di contesto.

---

````text
Costruiamo un function hook locale per Claude Code che sostituisce la
compattazione nativa del contesto con una compattazione SENZA riscrittura,
guidata da Jev (TypeSafe System One), e che parte a una soglia di contesto
scelta da me.

## Perché

La compattazione nativa sostituisce la conversazione con un riassunto scritto da
un modello. Tutto ciò che sopravvive è riscritto: un path, un numero di riga, il
testo esatto di un errore possono deformarsi senza che nessuno se ne accorga.

Questo hook fa invece una selezione binaria su contenuto originale: mostra a Jev
la struttura della conversazione, chiede per ogni tool call se serve ancora, e
poi elimina o tronca i `tool_result` che non servono più. Ciò che resta è
verbatim. I messaggi di testo di utente e assistente non vengono toccati mai.

In una sessione lunga i `tool_result` sono la gran parte dei token e la maggior
parte non serve più (il file letto e poi riscritto, il `ls` iniziale, il grep a
vuoto), mentre i messaggi che portano l'intento pesano pochi punti percentuali.

## Vincoli — cosa NON costruire

- nessun pacchetto npm, nessun `package.json`, nessuna dipendenza da installare
- nessun marketplace, nessun `claude plugin install`, nessun `--plugin-dir`
- nessuna app demo, GUI, cartella `examples/`
- nessun secondo hook oltre a `session.compact`
- nessun `userConfig` nel manifest: le opzioni sono costanti in cima al file
- nessun file di stato su disco, nessuna telemetria
- nessuna astrazione «per il futuro»: niente interfacce con una sola
  implementazione, niente strategy pattern

Il criterio: se lo tolgo e la compattazione funziona uguale, non va scritto.

## Passo 0 — leggi i tipi prima di scrivere una riga

L'API dei function hook è early access e cambia fra le release. Non indovinare
nessuna firma.

1. `claude --version` → serve ≥ 2.1.276. Se è più vecchio, fermati e dimmelo.
2. Genera le dichiarazioni del mio build:
   `CLAUDE_CODE_ENABLE_FUNCTION_HOOKS=1 claude -p "/plugin-types .claude/types"`
   e aggiungi `.claude/types/` al `.gitignore`.
3. In `.claude/types/claude-code.d.ts` leggi: `Register`, `SessionCompactInput`,
   `SessionCompactResult`, `SessionMessage`, `ToolUseSummary`,
   `ToolResultSummary`, `HookBudget`, `$.http.fetch`, `$.clock`, `$.env`,
   `$.ui.log`, `$.ui.status`, `$.ui.toast` e il modulo `claude-code/testing` in
   fondo al file.
4. Leggi la documentazione live di Jev: `https://docs.typesafe.ai/api.md` e
   `https://docs.typesafe.ai/primitives/noul.md`.

Quello che trovi nei tipi vince su qualunque cosa scritta in questo prompt.
Se una firma qui sotto non corrisponde, segui i tipi e dimmelo.

## Passo 1 — la soglia di contesto (chiedimela prima di creare i file)

L'hook non decide quando compattare: parte quando Claude Code emette
`session.compact`, cioè con `/compact` manuale o con l'auto-compact. La soglia
dell'auto-compact si imposta con la variabile `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE`,
in percentuale della finestra di contesto.

Chiedimi a quale percentuale di contesto usato deve partire la compattazione
(usa lo strumento per le domande all'utente se disponibile, con alcune opzioni
tipo 30, 50, 70 e la possibilità di scrivere un valore). Prima della domanda
spiegami in due righe il compromesso: una soglia bassa compatta spesso e tiene
il contesto leggero, una soglia alta compatta raramente e più a fondo; il valore
può solo anticipare la soglia nativa, non ritardarla. Accetta un intero tra 1 e
99; se rispondo con un valore non valido, richiedilo.

## Passo 2 — setup: tre file e il blocco env

Una cartella in `.claude/skills/` con un manifest viene adottata come plugin da
sola alla sessione successiva (`jev-compact@skills-dir`). Nessun flag, nessuna
installazione. (Chi la vuole attiva in ogni progetto sposta la stessa cartella
in `~/.claude/skills/` e il blocco `env` in `~/.claude/settings.json`.)

```
.claude/skills/jev-compact/
├── .claude-plugin/plugin.json     # { name, version, description, author } e basta
├── hooks/hooks.json               # { "modules": ["./compact.ts"] }
└── hooks/compact.ts               # tutta la logica, un solo file
```

Poi in `.claude/settings.json` del progetto, preservando tutto il resto
(leggilo, fai il merge, non sovrascriverlo), con la percentuale del Passo 1:

```json
{
  "env": {
    "CLAUDE_CODE_ENABLE_FUNCTION_HOOKS": "1",
    "CLAUDE_AUTOCOMPACT_PCT_OVERRIDE": "<percentuale scelta>"
  }
}
```

Se nei settings utente (`~/.claude/settings.json`) è presente
`CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`, aggiungi nello stesso blocco
`"CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC": ""` (stringa vuota: `"0"` non basta),
altrimenti `$.http.fetch` viene rifiutato.

Da non confondere con `MIN_REDUCTION` (sotto), che è la riduzione minima ottenuta.

La chiave API Jev si legge con `$.env.get("TYPESAFE_API_KEY")`: arriva dalla
shell oppure da `env` in `~/.claude/settings.json` o `.claude/settings.local.json`
(file locale, da tenere fuori da git). Mai nei sorgenti, nel manifest o in un
file versionato. Non chiedermi di incollarla in chat: se manca, dimmi dove metterla.

Un solo file TypeScript: `$` non attraversa un `import`, quindi tenere tutto in
`compact.ts` evita il problema in partenza. Il modulo gira in un ambiente
proprio senza Node né DOM: rete, env e clock passano solo da `$`. Ogni funzione
che riceve `$` deve essere una `function` al top-level del file, e `$.env.get`
va chiamato con il nome come stringa letterale (regole del validatore).

## Passo 3 — l'hook

Registra solo `session.compact`:

```ts
import type { Register } from "claude-code";

export const register: Register = (on) => {
  on("session.compact", async ($, e, next) => { /* ... */ });
};
```

- `e.messages` è la trascrizione da compattare, ogni messaggio col suo `handle`
- restituisci `{ messages }` con il nuovo array per installare la tua compattazione
- restituisci `next(e)` per delegare al riassunto nativo

Delega a `next(e)` senza esitare quando: manca la chiave, la chiamata a Jev
fallisce o va in timeout, la risposta non ha la forma attesa, non ci sono
candidati, oppure la riduzione stimata è sotto `MIN_REDUCTION`. Un fallback
silenzioso vale più di un tentativo eroico: se l'hook rompe la compattazione,
rompe la sessione.

`e.trigger === "precompute"` non installa niente (il risultato viene tenuto per
la compattazione che verrà): trattalo come gli altri. Se `e.agentId` è presente
stai compattando un subagent: non toccare quel campo.

Metti un timeout sulla chiamata a Jev (race fra `$.http.fetch` e
`$.clock.sleep`): il budget dell'hook (10 s) si ferma mentre la fetch è in volo,
quindi una fetch appesa non scade da sola. Le attese `$.clock` invece lo
consumano: il timeout deve stare sotto i 10 s, altrimenti scade prima il budget
e l'engine delega al nativo senza dirlo.

Output per l'utente, tutto via `$.ui`:
- barra di avanzamento con `$.ui.status(...)` a ogni fase — `jev-compact ███░░░░░░░ 30% · chiedo a Jev (1/3 batch, 40 call)`:
  10% ricerca candidate, 20→80% batch Jev man mano che rispondono, 90% ricostruzione
  (o `riassunto nativo` in fallback). Jev risponde in circa un secondo: la barra da
  sola sparirebbe prima di poterla leggere. Perciò a fine hook non si cancella
  subito: si fissa l'esito
  (`jev-compact ██████████ 100% · -26% (25.1k chars) · 5 troncate, 0 eliminate, 16 tenute`,
  oppure `Jev non usato (<motivo>) → riassunto nativo`) con `$.ui.status`, lo si ripete in
  `$.ui.toast(…, { timeoutMs: 8000 })` e lo si libera con
  `$.clock.after(RESULT_VISIBLE_MS = 30_000, () => $.ui.status(undefined))` — anche in
  fallback (`try/finally` attorno a `next(e)`), altrimenti la riga resta appesa.
  In headless non c'è status row: la riga finisce solo nel debug log.
- due righe `$.ui.log(...)` di riepilogo in chiaro: `Jev ha compattato il contesto: -26% (25.1k chars su 96.4k) in 1061ms`
  e `21 tool call valutate → 16 tenute, 5 troncate, 0 eliminate · messaggi 70 → 70`;
  in fallback `Jev non ha compattato (<motivo>): riassunto nativo di Claude Code`;
- sotto, una riga `$.ui.log` per ogni call toccata: `✗ eliminata  Bash · wc -l src/server.ts  (-120 chars)`
  o `✂ troncata   Read · docs/architecture.md  (16.8k → 300 chars)` — il tool e
  l'argomento che identifica la call (`file_path`, `command`, `pattern`, `url`,
  `query`; altrimenti il JSON), abbreviato a 70 caratteri, mai il JSON grezzo.
  Le call tenute non si elencano.

## Passo 4 — l'algoritmo, in ordine

1. Accoppia le chiamate. Scorri `e.messages`: `toolUses[]` sui messaggi
   `assistant` (`tool_use_id`, `tool`, `input`) e `toolResults[]` sui messaggi
   `user` (`tool_use_id`, `text`, `isError`). Appaia per `tool_use_id`. Una call
   senza result, o viceversa, non è un candidato.

2. Il primo messaggio e gli ultimi `PRESERVE_RECENT` sono immuni.

3. Costruisci lo state per Jev. L'intera conversazione, in ordine, come
   array di voci `{ i, role, text, tool_calls: [{ id, tool, input, result }] }`
   dove `result` è solo una nota tipo `"ok, 4212 chars"` / `"error, 80 chars"`.
   Così Jev vede struttura, ordine e cronologia senza pagare i token del
   contenuto. Aggiungi `goal`: gli ultimi 3 messaggi utente. Se lo state stimato
   supera `MAX_STATE_TOKENS` (stima: caratteri / 4), comprimilo per gradi:
   tronca gli input delle call, poi abbrevia i testi lunghi a testa+coda
   partendo dai più vecchi non protetti, poi collassa i messaggi più vecchi a
   una nota `[… N chars]`. Se non rientra, delega a `next(e)`.

4. Chiedi a Jev. Per ogni candidato due domande Noul (probabilità di sì):

   - `keep_call_{id}`: «La tool call `{id}` (`{tool}`) deve restare nella
     cronologia: sapere che è stata fatta, con il suo input, conta ancora per
     quello che l'assistente farà dopo.»
   - `keep_result_{id}`: «L'output completo della tool call `{id}` (`{tool}`,
     `{chars}` caratteri) deve restare verbatim: all'assistente serve ancora il
     contenuto e rieseguire il tool non basterebbe.»

   Gli ID delle domande non vengono visti dal modello: il significato completo
   sta nel testo. Raggruppa le domande in batch per restare sotto
   `MAX_REQUEST_TOKENS` (state + domande), rimandando lo state completo con ogni
   batch; i batch partono in parallelo.

5. Decidi, per ogni coppia.

   | keep_result | keep_call | azione |
   |---|---|---|
   | ≥ soglia | — | tieni tutto |
   | < soglia | ≥ soglia | tieni la call, tronca il result ai primi `TRUNCATE_HEAD` caratteri + `[… N chars rimossi]` — solo se il result è più lungo di `TRUNCATE_HEAD`, altrimenti tieni tutto (l'avviso lo allungherebbe) |
   | < soglia | < soglia | elimina la coppia, call e result insieme |

   Mai un `tool_use` orfano senza `tool_result`, né il contrario.
   Un messaggio che resta senza testo e senza blocchi viene rimosso.

6. Ricostruisci l'array nello stesso ordine e restituiscilo.

7. Controlla la riduzione sui caratteri: se `1 - dopo/prima < MIN_REDUCTION`,
   scarta il lavoro e fai `next(e)`.

## Passo 5 — il punto critico: `handle`

L'engine associa un `handle` opaco a ogni messaggio che passa all'hook. Quando
restituisci l'array:

- un messaggio con il suo handle viene preso come l'originale, intero: le
  modifiche al contenuto vengono ignorate
- un messaggio senza handle viene ricostruito da `role`, `text`, `toolUses`,
  `toolResults`

Quindi: togli l'handle a TUTTI i messaggi che restituisci, anche a quelli che non
hai toccato (`const { handle, ...rest } = message`). Un messaggio modificato ma
con handle non cambia nulla: sembra che l'hook non giri. Un messaggio intatto
restituito con handle viene ri-registrato con la catena dei parent di prima della
compattazione: alla `--resume` successiva il loader reinserisce anche i messaggi
vecchi (tool_use duplicati, warning `ensureToolResultPairing`). Senza handle su
tutti, la catena è pulita.

## Costanti

In cima a `compact.ts`:

| Costante | Default | Cosa fa |
|---|---|---|
| `KEEP_THRESHOLD` | `0.5` | Probabilità minima per tenere |
| `PRESERVE_RECENT` | `6` | Ultimi messaggi mai toccati |
| `TRUNCATE_HEAD` | `300` | Quanto resta di un result scartato |
| `MAX_STATE_TOKENS` | `25000` | Tetto dello state mandato a Jev |
| `MAX_REQUEST_TOKENS` | `30000` | Tetto di una singola richiesta |
| `MIN_REDUCTION` | `0.15` | Sotto questa riduzione, delega al nativo |
| `JEV_TIMEOUT_MS` | `8000` | Timeout di ogni richiesta a Jev (sotto il budget di 10 s dell'hook) |
| `RESULT_VISIBLE_MS` | `30000` | Per quanto resta visibile l'esito nella riga di stato |
| `JEV_MODEL` | `"jev-latest"` | Modello |

## Il contratto Jev

Verifica sulla pagina API live; riferimento:

- `POST https://api.typesafe.ai/v1/systemone`
- header `authorization: Bearer <chiave>`, `content-type: application/json`
- body `{ model, state, questions }`; `state` è un oggetto JSON
- ogni domanda: `{ type: "noul", instructions, criteria: { true, false } }`
- risposta `{ answers: { <id>: { noul: number } } }`; se `answers` manca o un
  `noul` non è un numero finito → fallimento → `next(e)`

## Verifica — non dichiarare fatto prima

1. `claude plugin validate .claude/skills/jev-compact` → deve elencare
   `session.compact` e le `$`-call usate.
2. `hooks/compact.test.ts` con il kit `claude-code/testing`, eseguito con
   `claude plugin test .claude/skills/jev-compact`. Copri: accoppiamento per id,
   le tre decisioni della tabella, nessun `handle` sui messaggi restituiti,
   fallback senza chiave, su risposta malformata, su HTTP non 2xx, su timeout
   e su riduzione insufficiente, nessun `tool_use` orfano.
   Nel test il mondo sotto il plugin va simulato: gli hook sui verbi `$`
   rispondono `{ value }` (es. `on('http.fetch', () => ({ value: { status: 200, ok: true, headers: {}, text } }))`),
   servono `mock.env(on, …)`, `mock.clock(on)` e `on('ui.log', () => ({ value: undefined }))`
   (lo stesso per `ui.status` e `ui.toast`), e il fondo della catena è
   `on('session.compact', ($, e) => ({ messages: e.messages }))` con un flag per
   sapere se l'hook ha delegato. L'hook riceve copie: verifica il contenuto, non
   l'identità degli oggetti. Se l'engine riporta «hook was skipped: no
   implementation for …», il test sta passando per il motivo sbagliato.
   Il timeout si prova con `mock.clock`: la fetch finta aspetta `clock.sleep(60_000)`,
   il test lancia la compattazione senza `await`, `clock.advance(9_000)`, poi verifica
   la delega.
3. Se la chiave è presente: sessione headless con qualche tool call, poi
   `/compact`, poi un altro turno e infine una `--resume` della stessa sessione.
   `claude -p --debug --session-id <uuid> "..."`, poi
   `echo /compact | claude -p --debug --resume <uuid>`, poi un terzo
   `claude -p --debug --resume <uuid> "..."`. Nel debug log
   (`~/.claude/debug/latest`) deve esserci la riga di riepilogo di jev-compact e
   nessun `ensureToolResultPairing`. La cartella `.claude/skills/` di un progetto
   viene letta solo se il progetto è "trusted": serve aver accettato una volta il
   prompt di fiducia aprendo `claude` in interattivo. Se non gira, controlla
   nell'ordine: l'`env` in `.claude/settings.json`, il path in `hooks.json`, il `handle`.

## Fatto quando

- `/compact` in una sessione vera produce una conversazione più corta in cui i
  messaggi di utente e assistente sono identici a prima, e alcuni tool result
  sono spariti o troncati; il turno dopo e la `--resume` funzionano senza warning
- l'auto-compact parte alla percentuale che ho scelto
- senza `TYPESAFE_API_KEY` la sessione compatta normalmente, senza errori visibili
- `claude plugin validate` e `claude plugin test` passano
- il codice sta in tre file e si legge in una passata
- non serve nessun flag né comando extra: apri `claude` e c'è

Quando hai finito, dimmi in poche righe: quali file hai creato, la soglia
impostata, l'output dei comandi di verifica, e quali scelte hai preso dove questo
prompt era ambiguo.
````
