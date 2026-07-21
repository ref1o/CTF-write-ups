# Cache Me Outside — Write-up

**Room:** Cache Me Outside (TryHackMe)
**Tema:** OSINT — investigazione su identità digitale
**Obiettivo:** Rintracciare un ex-hacker diventato outdoorsman ("Can you find this ex hacker turned outdoorsman?") e ricostruire, partendo da uno screenshot di una conversazione, gli indizi che ha lasciato online (nome, email, telefono, città, spostamenti).

**Punto di partenza:** screenshot di una chat tra `JJ ^_^` (il target) e `WKM1337?` (un vecchio contatto).

---

> 🔒 **Nota privacy:** questo write-up documenta la *metodologia* completa (la catena di pivot OSINT) ma **oscura i dati personali specifici** (`<REDACTED>`) — email, telefono, posizione, spostamenti. È intenzionale: le room OSINT chiedono di non diffondere i dati raccolti, e il valore didattico sta nel *metodo*, non nei valori. Le risposte in chiaro vanno inserite solo nei campi della room.

> ⚠️ **Disclaimer etico:** esercizio svolto su una room di training con un personaggio fittizio, in ambiente controllato e autorizzato. L'OSINT attivo contro persone reali senza autorizzazione è illegale e non etico.

---

## Level 1 — Profilo Komoot → nome reale

**Indizio dallo screenshot:**
> Il target dichiara gli hobby (hiking, cycling, running) e condivide spontaneamente il proprio profilo percorsi su **Komoot**: `komoot.com/user/<ID>`.

Indizi estratti direttamente dall'immagine:

| Indizio | Valore |
|---------|--------|
| Hobby dichiarati | hiking, cycling, running |
| Piattaforma usata per i percorsi | **Komoot** |
| Link profilo condiviso | `komoot.com/user/<ID>` |

### Passi

1. Aprire il profilo Komoot condiviso nella chat.
2. Leggere il nome visualizzato, la bio e i link ad altri account collegati.

### Risultato

- **Nome completo visualizzato:** `<REDACTED>` *(due parole — formato `*** ***`)*
- **Bio:** si descrive come ex-hacker che ha cambiato vita, ora dedito a running e outdoor — coerente con la chat.
- **Nuovo pivot esposto:** un link a **GitHub** (`github.com/<handle>`), con l'handle costruito come *nome + leetspeak*.

> ✅ **Domanda 1 (nome completo):** risolta dal profilo Komoot.

**Tecnica:** un profilo condiviso pubblicamente lega uno pseudonimo (`JJ ^_^`) a un nome reale e ad altri account collegati.

---

## Level 2 — GitHub → email esposta nei metadati dei commit

**Indizio (dal profilo GitHub):**
> Il profilo conferma l'identità ("Security Consulting | Ex-Hacker | Avid Runner") e presenta un **profile repository** con un `README.md`.

### Passi

1. Ispezionare il `README.md` del profile repository. È il template di default di GitHub, ma la riga auto-generata cita un **username diverso** da quello del profilo → rivela un secondo handle (`<REDACTED>`) da cui il README era stato copiato. Un "forgotten detail".

2. **Estrazione dell'email (tecnica forense su git).** La cronologia dei commit registra l'email dell'autore nei metadati. Aggiungendo `.patch` all'URL di un commit si ottiene l'header completo:

   ```
   From: <handle> <<REDACTED-email>>
   Date: Thu, 16 Apr 2026 03:27:19 -0400
   ```

### Risultato

- **Secondo handle esposto:** `<REDACTED>` — dal template README copiato.
- **Email esposta:** `<REDACTED>` *(formato `****************.***`)* — recuperata dai metadati del commit, non dal contenuto del profilo.
- **Bonus geolocalizzazione:** il fuso orario del commit (`-0400`) è un indizio sul fuso/area del target, utile per le domande su città e spostamenti.

> ✅ **Domanda 2 (email):** risolta dai metadati dei commit GitHub.

**Lezione appresa:** l'email personale "trapela" quasi sempre nei metadati git (`git log`, `.patch`), anche quando non è scritta da nessuna parte nel profilo. È l'errore di OPSEC più comune tra gli sviluppatori.

---

## Level 3 — Username enumeration → Threads (pivot)

**Indizio:** i due handle noti (Level 1 e 2) sono riusabili per cercare account con lo stesso nome su altre piattaforme.

### Passi

1. **Username enumeration** cross-platform (Sherlock / osintsearchengine / WhatsMyName) sui due handle noti.
2. Filtrare i match: la maggior parte sono **falsi positivi** (username comuni → piattaforme di intrattenimento non pertinenti). Il filtro è sempre la coerenza col personaggio (ex-hacker / runner / cyclist).
3. Seguire il pivot valido: un **profilo social** (display name coerente col nome reale) che rimanda a un account **Threads** attivo.

### Risultato

Threads, essendo testuale, è la miniera di dettagli quotidiani. **Post chiave (datato 07/05/2026):**

> *"Just finished my last run before the big day, hopping on the tram for my well-deserved coffee at my favourite French supermarket."* + **foto di una strada**.

Elementi OSINT nel post:
- conferma dell'uso del **tram** in quella data esatta (→ Domanda 5)
- riferimento a un **supermercato francese** (catena tipo Auchan/Carrefour/Decathlon) come punto di riferimento
- una **foto** con indizi geolocalizzabili

> Nota di design: questo Level non risponde da solo a una domanda, ma è il *pivot* che alimenta i Level 4 e 6.

**Lezione appresa:** riusare lo stesso username ovunque rende banale l'enumerazione cross-platform; i social testuali espongono luoghi, orari e abitudini.

---

## Level 4 — Geolocalizzazione della foto → città

**Indizio:** la foto del post del 07/05 contiene testo leggibile sullo sfondo.

### Passi

1. Insegna leggibile nella foto: **`IRIGATII.RO`** → dominio `.ro` = **Romania**.
2. `IRIGATII.RO` è un'**azienda reale**: ricerca del nome → scheda con indirizzo fisico.
3. Indirizzo: **Calea Buziașului 13, 300701 — Timișoara**.

### Risultato

**Città = Timișoara** (9 lettere, combacia col formato richiesto; città rumena con rete tranviaria STPT).

> ✅ **Domanda 4 (città):** Timișoara

**Tecnica:** geolocalizzazione da immagine tramite testo visibile (insegna) + lookup dell'azienda su mappa. Un singolo dettaglio leggibile (un dominio `.ro` su un capannone) ancora l'intera posizione.

---

## Level 5 — Active OSINT: out-of-office → telefono

**Indizio (briefing della room):** la room segnalava un esempio di **active OSINT** (interazione con l'infrastruttura scoperta).

### Passi

1. Inviare un'email all'indirizzo trovato al Level 2.
2. Il target risponde con un **auto-reply di assenza** (out-of-office) contenente la **firma email completa**.

### Risultato

- **Ruolo confermato:** "Cybersecurity Consultant — L33T Security (Pentesting · Red Team · Consulting)"
- **Telefono esposto:** `<REDACTED>` — prefisso **+40 (Romania)**, coerente con Timișoara. *(formato `*** *** *** ***`)*

> ✅ **Domanda 3 (telefono):** risolta via out-of-office auto-reply.

**Tecnica:** l'active OSINT (interazione diretta) può innescare risposte automatiche che espongono dati non presenti nelle fonti passive.

> ⚠️ **Attenzione etica/operativa:** interagire con un target reale può allertarlo ed è lecito solo con autorizzazione — qui è parte del setup controllato della room.

---

## Level 6 — Fermata del tram (7 maggio 2026)

**Indizio:** la foto è scattata lungo **Calea Buziașului** (strada 592); il post cita la discesa dal **tram** per un caffè al **supermercato francese** preferito.

### Passi

1. Lungo Calea Buziașului si identifica il supermercato francese: un **Auchan**.
2. La fermata del tram adiacente all'Auchan è **Piața Gheorghe Domășneanu** (rete STPT Timișoara).
3. **Validazione con la maschera** della risposta (`***** ******** **********` = 5 + 8 + 10 caratteri):
   - `Piața` (5) · `Gheorghe` (8) · `Domășneanu` (10) → combacia esattamente.

### Risultato

Coerenza narrativa: la fermata di discesa per il "French supermarket" è quella dell'Auchan, non quella più vicina al punto della foto (che era lungo il tragitto).

> ✅ **Domanda 5 (fermata tram):** Piața Gheorghe Domășneanu
> *(verificare grafia con/senza diacritici a seconda di ciò che la room accetta)*

**Tecnica:** incrocio tra indizio testuale (supermercato francese), geolocalizzazione della via, mappa dei trasporti pubblici e validazione tramite la maschera del formato di risposta.

---

## Riepilogo finale

| # | Domanda | Fonte / Tecnica | Risposta |
|---|---------|-----------------|----------|
| 1 | Nome completo | Profilo Komoot condiviso in chat | `<REDACTED>` |
| 2 | Email | Metadati commit GitHub (`.patch`) | `<REDACTED>` |
| 3 | Telefono | Active OSINT — out-of-office auto-reply | `<REDACTED>` |
| 4 | Città | Geolocalizzazione foto (insegna `.ro` + lookup azienda) | Timișoara |
| 5 | Fermata tram (7/5/2026) | Threads + supermercato francese + mappa STPT | `<REDACTED>` |

### Catena di pivot completa

```
Screenshot chat
   └─ profilo Komoot ──► nome reale + link GitHub
        └─ GitHub ──► 2° handle (template README) + email (metadati commit)
             └─ username enumeration ──► Threads
                  └─ Threads (post 07/05) ──► foto + "French supermarket" + tram
                       └─ geoint foto (IRIGATII.RO) ──► Timișoara, Calea Buziașului
                            └─ mappa STPT + Auchan ──► fermata del tram
        └─ email ──► active OSINT (out-of-office) ──► telefono
```

Ogni Level espone il successivo: è il cuore dell'OSINT investigativo — nessun dato è isolato, ognuno è una porta verso il prossimo.

### Lezioni di OPSEC (lato difensivo)

1. **Non legare pseudonimo e identità reale** condividendo profili pubblici.
2. **I metadati git contengono la tua email** — usare l'email `noreply` di GitHub.
3. **Riusare lo stesso username** ovunque rende banale l'enumerazione cross-platform.
4. **Le app di fitness/percorsi e i social testuali** espongono luoghi, orari e abitudini.
5. **Le foto rivelano la posizione** anche solo da un'insegna o un dominio sullo sfondo (geoint).
6. **Gli auto-reply (out-of-office)** possono esporre firma, telefono e ruolo a chiunque scriva.
7. **Piccoli dettagli dimenticati** (template copiato, fuso orario, vecchio handle) ricostruiscono l'identità completa.

---

*Write-up prodotto a scopo didattico su room di training OSINT (TryHackMe), in ambiente autorizzato. Dati personali oscurati intenzionalmente.*
