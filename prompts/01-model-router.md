# 01 — Model router

Costruisce un function hook che, al **primo messaggio** di ogni sessione, chiede a Jev quale modello e quale livello di effort servono per il task, e li mantiene per il resto della sessione. I subagenti vengono classificati singolarmente quando partono.

Il modello non cambia a ogni messaggio perché un cambio di modello o di effort invalida la prompt cache: ogni turno successivo rileggerebbe l'intera conversazione al prezzo pieno di input.

**Prerequisiti:** Claude Code ≥ 2.1.276, una API key TypeSafe (<https://console.typesafe.ai/keys>) in `TYPESAFE_API_KEY`.

**Uso:** apri `claude` nella cartella del progetto, incolla il prompt qui sotto e lascia lavorare Claude fino al riepilogo finale.

---

````text
Voglio un function hook locale per Claude Code che usi Jev, il modello decisionale
di TypeSafe, per scegliere il modello e il livello di effort con cui lavorare.
La scelta avviene UNA volta per sessione, sul primo prompt, e resta fissa per tutti
i turni successivi: cambiare modello o effort a metà conversazione invalida la
prompt cache e ogni turno rileggerebbe la conversazione a prezzo pieno. I subagenti,
che partono con un contesto proprio, vengono invece classificati uno per uno.

Costruiscilo interamente dentro questo progetto: niente plugin da installare,
niente pacchetti npm, niente flag da passare a `claude`. Deve caricarsi da solo
aprendo `claude` nella cartella.

## Come funziona il caricamento (non inventare alternative)

Claude Code carica automaticamente, da `.claude/skills/<nome>/`, ogni cartella che
contiene `.claude-plugin/plugin.json` e `hooks/hooks.json`. Il file `hooks.json`
elenca in `modules` un modulo TypeScript che esporta `register(on, options)`.
Ogni hook ha la forma `($, e, next)`: `$` è l'interfaccia del motore
(`$.http.fetch`, `$.env.get`, `$.ui.log`, `$.ui.status`, `$.clock`...), `e` è
l'input dell'evento, `next(e)` prosegue la catena e `next({ ...e, campo })` riscrive
ciò che il motore vede. L'evento `turn.step` è streaming: il suo hook è un
`async function*` che fa `return yield* next(e)`.

I function hook si attivano con la variabile d'ambiente
`CLAUDE_CODE_ENABLE_FUNCTION_HOOKS=1`. Mettila in `.claude/settings.json` del
progetto sotto `env`, così non serve nessun flag.

Prima di scrivere codice:
1. `claude --version` → serve ≥ 2.1.276. Se è più vecchio, fermati e dimmelo.
2. Genera le dichiarazioni dei tipi del mio build con
   `CLAUDE_CODE_ENABLE_FUNCTION_HOOKS=1 claude -p "/plugin-types .claude/types"`
3. Leggi in `.claude/types/claude-code.d.ts` gli input degli eventi
   `session.start`, `prompt.submit`, `turn.step`, `agent.spawn` e i metodi di `$`
   che userai. Non tirare a indovinare campi o firme: quello che trovi nei tipi
   vince su questo prompt. Se qualcosa qui sotto non corrisponde, segui i tipi e
   dimmelo nel riepilogo.
4. Aggiungi `.claude/types/` al `.gitignore`.

## File da creare

- `.claude/settings.json` → `{ "env": { "CLAUDE_CODE_ENABLE_FUNCTION_HOOKS": "1" } }`.
  Se il file esiste già, leggilo e aggiungi solo la chiave: non sovrascrivere il
  resto. Se nei settings utente (`~/.claude/settings.json`) è presente
  `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`, aggiungi nello stesso blocco
  `"CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC": ""` (stringa vuota: `"0"` non basta),
  altrimenti `$.http.fetch` viene rifiutato con
  "nonessential network traffic is disabled".
- `.claude/skills/jev-router/.claude-plugin/plugin.json` → `name` `jev-router`,
  `version`, `description`, `author` (senza `author` il validatore avvisa). Nessun
  `userConfig`: tutta la configurazione vive come costanti in cima al modulo.
- `.claude/skills/jev-router/hooks/hooks.json` → `{ "modules": ["./router.ts"] }`.
- `.claude/skills/jev-router/hooks/router.ts` → il modulo.
- `.claude/skills/jev-router/tsconfig.json` → quello nell'header di
  `claude-code.d.ts`, con `"include": ["../../types", "hooks"]`.
- `.claude/skills/jev-router/README.md` → mezza pagina: cosa fa, dove va la chiave,
  come si legge il log, come si spegne (cancellare la cartella).

Regole del validatore (`claude plugin validate`): ogni funzione che riceve `$`
deve essere una `function` dichiarata al top-level del file (non una closure
dentro `register`), e `$.env.get` va chiamato con il nome come stringa letterale.
Lo stato condiviso tra hook (chiave letta, decisione della sessione) va in
variabili di modulo o dentro `register`, ma le funzioni che usano `$` restano fuori.

## La chiave API

`await $.env.get("TYPESAFE_API_KEY")`. Arriva dall'ambiente della shell oppure da
`env` in `~/.claude/settings.json` o in `.claude/settings.local.json` (file locale,
da tenere fuori da git). Mai nei sorgenti, nel manifest o in un file versionato.
Non chiedermi di incollarla in chat: se manca, dimmi dove metterla.
Senza chiave il router scrive UNA riga di log alla prima esecuzione e non tocca nulla.

## Cosa fa il modulo

1. `prompt.submit`: se la sessione non ha ancora una decisione, manda il testo del
   prompt a Jev e conserva la decisione. Poi `return next(e)`. Dal secondo prompt
   in poi non chiama Jev: una sola chiamata per sessione. Una risposta mancata
   (timeout, errore) non conta come decisione: si riprova al prompt successivo.

   Richiesta: `POST https://api.typesafe.ai/v1/systemone`, header
   `authorization: Bearer <key>` e `content-type: application/json`. Il body è
   ESATTAMENTE questo (usa questi nomi di campo, non inventarne altri: ogni domanda
   ha `type`, `instructions`, `criteria`):

   ```json
   {
     "model": "jev-latest",
     "state": { "prompt": "<testo del prompt>" },
     "questions": {
       "tier": {
         "type": "choice",
         "instructions": "Which is the cheapest tier that can complete this coding task well?",
         "criteria": {
           "fast": "Mechanical and local: read or summarise a file, run one command, rename a symbol, answer something already in context.",
           "balanced": "Ordinary engineering: implement a well-specified change across a few files, write tests, fix a clearly described bug, review a small diff.",
           "deep": "Hard or high-stakes: architecture and design, debugging a failure whose cause is unknown, security, data migrations, concurrency, anything touching production or money."
         }
       },
       "effort": {
         "type": "score",
         "instructions": "How much step-by-step reasoning does this task need?",
         "criteria": ["almost none", "some", "a lot", "as much as possible"]
       },
       "risky": {
         "type": "noul",
         "instructions": "Carrying out this task would itself change production, move real money, or alter data that cannot be restored. Writing or testing code that deals with such things, without running it against the real system, does not count."
       }
     }
   }
   ```

   Jev non deve mai vedere nomi di modelli: sceglie tra descrizioni del lavoro. I
   quattro livelli di `effort` mappano su `low / medium / high / xhigh`.

   Risposta: `answers.tier.choice` + `answers.tier.confidence`,
   `answers.effort.score` (0..3) + `answers.effort.confidence`,
   `answers.risky.noul` (0..1).
   Timeout 1500 ms via `Promise.race` con `$.clock.sleep` (Jev risponde in
   300-800 ms). Qualsiasi errore (HTTP non 2xx, JSON illeggibile, timeout,
   eccezione) → nessuna decisione, mai un'eccezione fuori dall'hook.

2. `turn.step` (solo loop principale, cioè quando `e.agentId` è assente): applica
   la decisione della sessione a ogni step, sempre la stessa:
   - `effort`: se supera la soglia della policy.
   - `model`: se supera la soglia della policy. Passa l'id completo, non l'alias,
     con la costante `MODEL_ID` in cima al modulo:
     `sonnet → claude-sonnet-5`, `opus → claude-opus-5-5`, `fable → claude-fable-5-1`.
     Se il mio account o il mio build non accettano uno di questi id (controlla
     con `/model` o nei tipi), usa quelli disponibili e dimmelo.
   Finché non c'è una decisione, lascia passare `e` intatto.

3. `session.start` (se l'evento esiste nei tipi): azzera la decisione, così una
   nuova sessione o un `/clear` ripartono con una classificazione nuova.

4. `agent.spawn`: classifica `{ prompt, description, agentType }` del subagente con
   la stessa richiesta e imposta `model` con l'alias del tier. I tre tier mappano
   così: `fast → sonnet`, `balanced → opus`, `deep → fable` (costante `TIER_ALIAS`
   in cima al modulo, così chi non ha accesso a un modello la cambia in una riga).
   Salta i fork (`e.fork === true`, ereditano sempre). Il confronto parte da
   `e.model ?? e.parentModel`. Ogni subagente ha la sua chiamata a Jev: il suo
   contesto è nuovo, quindi non c'è cache da proteggere.

## Policy (in codice, non nelle domande)

- Salire di livello richiede confidence ≥ 0.3; scendere richiede ≥ 0.6 (sbagliare
  in su costa centesimi, sbagliare in giù affida un task serio a un modello piccolo).
- `risky` > 0.7 → tier `deep` e effort almeno `high`, saltando le soglie; il
  rischio alza il pavimento ma non abbassa mai un effort già più alto.
- Un `effort` numerico o un modello che non corrisponde a nessun tier: si può solo
  salire, mai scendere.
- Stessa decisione del valore corrente = nessun cambiamento.

## Log

Con `$.ui.log` scrivi una riga `[jev-router] pronto ...` alla prima esecuzione solo
nel debug log (`{ to: "debug" }`); l'avviso di chiave assente va invece nel
transcript. Nel transcript va UNA riga quando la decisione della sessione viene
presa e una per subagente, solo modello ed effort con cui parte davvero la richiesta:

```
[jev-router] sessione: sonnet · effort low
[jev-router] subagente Explore: sonnet
```

Tutto il resto va nel debug log con `{ to: "debug" }`: la risposta di Jev
(`classificazione (prompt): fast → sonnet (0.98) · effort low (0.94) · rischio 0.04 · 712ms`),
cosa è cambiato con il motivo (`modello opus → sonnet, effort medium → low (fast,
confidence 0.98)` oppure `invariato — ...`) e, dai prompt successivi,
`riuso la scelta del primo prompt`. Solo testo piano: le righe di log le disegna il
motore in grigio e i codici ANSI verrebbero stampati come caratteri. La riga di stato
`jev · sonnet · effort low` via `$.ui.status` è dietro la costante `SHOW_STATUS`, `true`.

## Verifica — non dichiarare fatto prima

- `claude plugin validate .claude/skills/jev-router` deve elencare gli eventi
  registrati e passare.
- Type-check: `npx -y -p typescript@5 tsc -p .claude/skills/jev-router/tsconfig.json`
  (ignora eventuali errori dentro `claude-code-mcp.d.ts`).
- Se nei tipi c'è il kit `claude-code/testing`, scrivi `hooks/router.test.ts` ed
  eseguilo con `claude plugin test .claude/skills/jev-router`. Copri: Jev chiamato
  una sola volta su due prompt consecutivi, stessa decisione applicata a tutti gli
  step, nuovo tentativo dopo un timeout, nessuna chiamata senza chiave, subagente
  classificato a ogni spawn.
- La cartella `.claude/skills/` di un progetto viene letta solo se il progetto è
  "trusted": serve aver accettato una volta il prompt di fiducia aprendo `claude` in
  interattivo. Se la chiave è presente, fai una prova headless
  (`claude -p --debug "leggi il README e riassumilo"`) e controlla nel debug log
  (`~/.claude/debug/latest`) `hooks module jev-router loaded` e la riga di
  classificazione. Ogni `claude -p` è un processo e una sessione a sé, quindi il
  riuso della scelta sui prompt successivi si verifica con il test del kit, oppure
  lo verifico io in una sessione interattiva: dimmi cosa guardare.
- Chiudi con un riepilogo: file creati, esito di ogni verifica, cosa manca (es. la
  chiave) e le scelte prese dove questo prompt era ambiguo.
````
