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
    └── Operation-Checkmate/
        ├── README.md          # write-up completo
        └── assets/            # screenshot e immagini (opzionale)
```

I write-up sono organizzati per **piattaforma** → **room/sfida**. Ogni sfida ha il proprio `README.md` che si renderizza automaticamente aprendo la cartella su GitHub.

---

## 📝 Write-up disponibili

| Piattaforma | Sfida | Categoria | Difficoltà |
|-------------|-------|-----------|------------|
| TryHackMe | [Operation Checkmate](./TryHackMe/Operation-Checkmate/) | Password Auditing / OSINT | Easy |

*(altri in arrivo)*

---

## 🛠️ Tecniche & strumenti ricorrenti

Enumerazione e ricognizione: `nmap`, `ffuf`, `gobuster`, `curl` ·
Attacchi a password: `hydra`, `cupp`, `john`, `hashcat` ·
OSINT e analisi web · reverse di hash · pivoting tra servizi

---

## 👤 Autore

**ref1o** · [GitHub](https://github.com/ref1o)

Se un write-up ti è stato utile, lascia una ⭐ al repo.
