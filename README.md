# CTF Write-ups

Raccolta personale di write-up per sfide **Capture The Flag** e room di training, a scopo didattico.
Ogni write-up documenta la metodologia, gli strumenti e il ragionamento dietro la soluzione — più che la semplice risposta.

> ⚠️ **Disclaimer**: tutto il contenuto è a scopo educativo e svolto in ambienti di laboratorio autorizzati (piattaforme di training o macchine di proprietà). Usa queste tecniche in modo etico e legale, solo su sistemi per cui hai esplicita autorizzazione.

---

## 📂 Struttura

```
CTF-write-ups/
├── README.md
└── TryHackMe/
    ├── Operation-Checkmate/
    │   └── README.md          # password auditing (5 livelli)
    ├── Cache-Me-Outside/
    │   └── README.md          # OSINT investigativo (catena di pivot)
    └── Bricks-Heist/
        └── README.md          # web RCE + incident response (miner/LockBit)
```

I write-up sono organizzati per **piattaforma** → **room/sfida**. Ogni sfida ha il proprio `README.md` che si renderizza automaticamente aprendo la cartella su GitHub.

---

## 📝 Write-up disponibili

| Piattaforma | Sfida | Categoria | Difficoltà |
|-------------|-------|-----------|------------|
| TryHackMe | [Operation Checkmate](./TryHackMe/Operation-Checkmate/) | Password Auditing / OSINT | Easy |
| TryHackMe | [Cache Me Outside](./TryHackMe/Cache-Me-Outside/) | OSINT / Investigazione | Medium |
| TryHackMe | [TryHack3M: Bricks Heist](./TryHackMe/Bricks-Heist/) | Web RCE / Incident Response | Medium |

*(altri in arrivo)*

---

## 🛠️ Tecniche & strumenti ricorrenti

**Enumerazione e ricognizione:** `nmap`, `ffuf`, `curl`
**Web exploitation:** RCE da CVE note (es. CVE-2024-25600 su Bricks Builder), reverse shell
**Attacchi a password:** `hydra`, `cupp`, wordlist mirate
**Reverse engineering:** unpack PyInstaller (`pyinstxtractor-ng`, `decompyle3`), deoffuscamento (ROT13 / base64)
**OSINT:** username enumeration (`sherlock`, WhatsMyName), geolocalizzazione da immagini (geoint), analisi metadati (git `.patch`), active OSINT (out-of-office)
**Analisi:** reverse di hash SHA256, attribuzione on-chain (blockchain explorer), pivoting tra servizi e piattaforme

---

## 🔒 Nota su privacy e spoiler

I write-up a tema OSINT **oscurano i dati personali** (`<REDACTED>`) e quelli password-based mostrano le tecniche mantenendo lo spirito della sfida. L'obiettivo è documentare il **metodo**, non fornire soluzioni pronte o esporre PII.

---

## 👤 Autore

**ref1o** · [GitHub](https://github.com/ref1o)

Se un write-up ti è stato utile, lascia una ⭐ al repo.
