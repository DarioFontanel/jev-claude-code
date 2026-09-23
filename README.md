# jev-claude-code

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](./LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude_Code-%E2%89%A5_2.1.276-d97757)](https://docs.anthropic.com/en/docs/claude-code)
[![Jev](https://img.shields.io/badge/TypeSafe-Jev-111111)](https://typesafe.ai/)

**jev-claude-code** è una raccolta di tre prompt per Claude Code che ti permettono di integrare Jev, il modello decisionale di TypeSafe AI, nel tuo flusso di lavoro: scelta del modello, compattazione del contesto e code review. Ogni prompt si incolla in una sessione di Claude Code e costruisce il sistema completo nel tuo progetto, verifiche comprese, senza pacchetti da installare.

---

## Perché Jev

Jev non è un LLM: non genera testo, risponde a domande chiuse con probabilità calibrate, in poche centinaia di millisecondi e a una frazione del costo di un modello di frontiera. Supporta tre tipi di domanda:

- **noul** — sì o no, con la probabilità del sì
- **choice** — una scelta fra opzioni che definisci tu, con la confidence
- **score** — una posizione su una scala che definisci tu

In tutti e tre i prompt Jev prende la decisione, mentre il codice applica le soglie e agisce. Ogni risultato si può ricondurre a un numero e a una regola leggibile.

---

## I tre sistemi

**🧭 01 — Model router** ([prompt](prompts/01-model-router.md)) — un function hook che, al primo messaggio di ogni sessione, chiede a Jev quale modello (Sonnet, Opus o Fable) e quale livello di effort servono per il task, e li mantiene per tutta la sessione. La scelta non si ripete a ogni messaggio perché un cambio di modello o di effort invalida la prompt cache. Ogni subagente viene classificato quando parte, dato che lavora con un contesto proprio.

**🗜️ 02 — Context compaction** ([prompt](prompts/02-context-compaction.md)) — un function hook che sostituisce la compattazione nativa. Invece di riassumere la conversazione, chiede a Jev quali tool call servono ancora ed elimina o tronca le altre: i messaggi tuoi e di Claude restano identici, i contenuti tenuti restano verbatim. Durante il setup ti viene chiesta la percentuale di contesto a cui far partire la compattazione.

**🔍 03 — Code review** ([prompt](prompts/03-code-review.md)) — un revisore in Python che invia un diff a Jev con 14 domande in una sola chiamata e calcola il verdetto (BLOCK, SECURITY REVIEW, NITS, MERGE) da soglie dichiarate in un file JSON. Include la skill `/jev-review`: Claude riporta il verdetto e legge il codice solo nei punti che Jev ha lasciato incerti.

```
prompt / diff ──▶ Jev (domande tipizzate) ──▶ probabilità ──▶ soglie nel codice ──▶ azione
```

---

## Prerequisiti

**Claude Code ≥ 2.1.276** — i prompt 01 e 02 usano i function hook, una funzionalità in early access che si attiva con la variabile `CLAUDE_CODE_ENABLE_FUNCTION_HOOKS=1`. Il prompt la imposta nei settings del progetto. Poiché l'interfaccia può cambiare fra le release, ogni prompt fa generare i tipi del tuo build e li considera prioritari rispetto al proprio testo.

**API key TypeSafe** — si crea su [console.typesafe.ai](https://console.typesafe.ai/keys). L'accesso è in early access: dopo la richiesta può servire fino a un giorno.

**Python 3.10+ e git** — solo per il prompt 03, senza dipendenze esterne.

---

## Installazione

1. Imposta la chiave in `TYPESAFE_API_KEY`, nella shell oppure sotto `env` in `~/.claude/settings.json`:

```bash
export TYPESAFE_API_KEY="la-tua-chiave"
```

2. Apri Claude Code nella cartella del progetto in cui vuoi il sistema — `claude`
3. Apri il file del prompt che ti interessa, copia il blocco di testo e incollalo nella sessione.
4. Segui Claude fino al riepilogo finale: crea i file, esegue le verifiche e ti segnala cosa manca.

Verifica: chiudi e riapri `claude` nella cartella. Il router scrive nel transcript una riga `[jev-router] sessione: …` al primo prompt; la compattazione mostra la riga `jev-compact` al primo `/compact`; la code review risponde a `/jev-review`.

I prompt sono indipendenti: puoi installarne uno, due o tutti e tre nello stesso progetto. La cartella `.claude/skills/` di un progetto viene caricata solo se hai accettato almeno una volta il prompt di fiducia aprendo `claude` in modalità interattiva. Per disattivare un sistema basta cancellare la sua cartella in `.claude/skills/`.

---

## Struttura del progetto

```
prompts/
├── 01-model-router.md          # scelta di modello ed effort al primo messaggio
├── 02-context-compaction.md    # compattazione del contesto con soglia configurabile
└── 03-code-review.md           # revisore del diff + skill /jev-review
```

Dentro ogni file trovi una breve descrizione, i prerequisiti e il prompt completo.

---

## Sicurezza

La chiave API si legge solo dall'ambiente, mai da un file versionato, e nessun prompt ti chiede di incollarla in chat. Il router e la compattazione, se la chiave manca o Jev non risponde, lasciano Claude Code al suo comportamento nativo. Il code review non modifica il codice: riporta il verdetto e chiede conferma prima di qualsiasi correzione.

---

Designed by **[Dario Fontanel, PhD](https://dariofontanel.com/)**

*Aiuto PMI italiane ad integrare l'intelligenza artificiale per automatizzare i lavori ripetitivi, abbattere i costi e guadagnare tempo per crescere.*

[![Sito](https://img.shields.io/badge/Sito-dariofontanel.com-4285F4?style=flat&logo=googlechrome&logoColor=white)](https://dariofontanel.com/)
[![YouTube](https://img.shields.io/badge/YouTube-FF0000?style=flat&logo=youtube&logoColor=white)](https://www.youtube.com/@dariofontanel)
[![Instagram](https://img.shields.io/badge/Instagram-E4405F?style=flat&logo=instagram&logoColor=white)](https://www.instagram.com/dariofontanel.ai/)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-0A66C2?style=flat&logo=data%3Aimage%2Fsvg%2Bxml%3Bbase64%2CPHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHZpZXdCb3g9IjAgMCAyNCAyNCI%2BPHBhdGggZmlsbD0id2hpdGUiIGQ9Ik0yMC40NDcgMjAuNDUyaC0zLjU1NHYtNS41NjljMC0xLjMyOC0uMDI3LTMuMDM3LTEuODUyLTMuMDM3LTEuODUzIDAtMi4xMzYgMS40NDUtMi4xMzYgMi45Mzl2NS42NjdIOS4zNTFWOWgzLjQxNHYxLjU2MWguMDQ2Yy40NzctLjkgMS42MzctMS44NSAzLjM3LTEuODUgMy42MDEgMCA0LjI2NyAyLjM3IDQuMjY3IDUuNDU1djYuMjg2ek01LjMzNyA3LjQzM2MtMS4xNDQgMC0yLjA2My0uOTI2LTIuMDYzLTIuMDY1IDAtMS4xMzguOTItMi4wNjMgMi4wNjMtMi4wNjMgMS4xNCAwIDIuMDY0LjkyNSAyLjA2NCAyLjA2MyAwIDEuMTM5LS45MjUgMi4wNjUtMi4wNjQgMi4wNjV6bTEuNzgyIDEzLjAxOUgzLjU1NVY5aDMuNTY0djExLjQ1MnpNMjIuMjI1IDBIMS43NzFDLjc5MiAwIDAgLjc3NCAwIDEuNzI5djIwLjU0MkMwIDIzLjIyNy43OTIgMjQgMS43NzEgMjRoMjAuNDUxQzIzLjIgMjQgMjQgMjMuMjI3IDI0IDIyLjI3MVYxLjcyOUMyNCAuNzc0IDIzLjIgMCAyMi4yMjUgMHoiLz48L3N2Zz4%3D)](https://www.linkedin.com/in/dario-fontanel/)
[![TikTok](https://img.shields.io/badge/TikTok-000000?style=flat&logo=tiktok&logoColor=white)](https://www.tiktok.com/@dario.fontanel)
[![AI Academy](https://img.shields.io/badge/AI_Academy-E7514F?style=flat&logo=data%3Aimage%2Fsvg%2Bxml%3Bbase64%2CPHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHZpZXdCb3g9IjAgMCAyNCAyNCI%2BPHBhdGggZmlsbD0id2hpdGUiIGQ9Ik0xMiAzIDEgOWwxMSA2IDktNC45MVYxN2gyVjlMMTIgM3pNNSAxMy4xOFYxN2MwIDEuNjYgMy4xMyAzIDcgM3M3LTEuMzQgNy0zdi0zLjgybC03IDMuODItNy0zLjgyeiIvPjwvc3ZnPg%3D%3D)](https://www.skool.com/ai-academy-2306)

Licenza [MIT](./LICENSE).
