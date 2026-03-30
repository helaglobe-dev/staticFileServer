# Static Files Stack — Helaglobe

Stack produzione per servire file statici con nginx e tunnel Pangolin, indipendente dallo stack Odoo.

## Architettura

```
Internet (HTTPS)
      │
 [Pangolin Server]
      │ tunnel WireGuard cifrato
      ▼
 [Newt container]
      │
      ▼
 [nginx-static container]  ←── security headers, cache, blocco path traversal
      │
      ▼
 /srv/static  (volume read-only)
```

## Struttura

```
.
├── docker-compose.static.yml   # Stack completo
├── nginx-static/
│   └── default.conf            # Virtual host nginx
└── static-files/               # File da servire (gitignored se contengono dati sensibili)
```

## Setup iniziale

### 1. Crea le cartelle necessarie

```bash
mkdir -p nginx-static static-files
```

### 2. Configura Newt
In `docker-compose.static.yml` imposta le credenziali Pangolin dedicate a questo stack:
```yaml
environment:
  - NEWT_ID=il-tuo-newt-id-static
  - NEWT_SECRET=il-tuo-newt-secret-static
```

> ⚠️ Usa credenziali Newt **separate** rispetto allo stack Odoo — ogni stack ha il proprio tunnel.

### 3. Posiziona i file statici

```bash
cp -r /percorso/tuoi/file/* static-files/
```

### 4. Avvia lo stack

```bash
docker compose -f docker-compose.static.yml up -d
```

### 5. Verifica

```bash
docker compose -f docker-compose.static.yml ps
docker logs nginx-static --tail 20
docker logs newt-static --tail 20
```

## Sicurezza

| Feature | Stato |
|---|---|
| Porte esposte su internet | ✅ Nessuna (tunnel Pangolin) |
| TLS/HTTPS | ✅ Pangolin (Let's Encrypt) |
| Security Headers | ✅ nginx |
| Accesso file nascosti (.env, .git…) | ✅ Bloccato (403) |
| Directory listing | ✅ Disabilitato |
| Volume static-files | ✅ Read-only |
| Rete Docker isolata | ✅ static_tunnel_net |
| Log rotation | ✅ json-file con limiti |
| Restart automatico | ✅ always |

## Comandi utili

```bash
# Stato container
docker compose -f docker-compose.static.yml ps

# Log in tempo reale
docker logs -f nginx-static
docker logs -f newt-static

# Ricarica nginx senza downtime
docker exec nginx-static nginx -s reload

# Aggiorna i file statici (senza downtime)
cp -r /nuovi/file/* static-files/
# nginx serve subito i nuovi file, nessun riavvio necessario

# Update immagini
docker compose -f docker-compose.static.yml pull
docker compose -f docker-compose.static.yml up -d
```

## Manutenzione

### Aggiornare i file statici
Basta copiare i nuovi file nella cartella `static-files/` — nginx li serve immediatamente senza riavvio grazie al volume live.

### Aggiornare nginx
```bash
docker compose -f docker-compose.static.yml pull nginx-static
docker compose -f docker-compose.static.yml up -d nginx-static
```

### Rinnovare certificato
Gestito automaticamente da Pangolin.

### Verificare i file serviti
```bash
docker logs nginx-static 2>&1 | grep -E "GET|POST|404|403" | tail -20
```

## Note

- **Non committare** i file statici se contengono dati sensibili o asset proprietari
- Le credenziali Newt vanno gestite tramite variabili d'ambiente o secrets Docker
- Questo stack è completamente **indipendente** dallo stack Odoo — si avvia, aggiorna e spegne separatamente
- La rete `static_tunnel_net` è isolata: nginx-static non può raggiungere Odoo o PostgreSQL