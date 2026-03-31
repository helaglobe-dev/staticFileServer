# Static Files Stack — Helaglobe

Stack produzione per servire file statici con nginx, FileBrowser e tunnel Pangolin, indipendente dallo stack Odoo.

## Architettura

```
Internet (HTTPS)
      │
 [Pangolin Server]
      ├── tunnel WireGuard (fileserver.helaglobe.com)
      │         │
      │   [newt-static] → [nginx-static] → /srv/static (read-only)
      │
      └── tunnel WireGuard (filebrowser.helaglobe.com)
                │
          [newt-filebrowser] → [filebrowser] → /srv (scrivibile)
```

I due container condividono lo stesso volume `static-files`: FileBrowser scrive, nginx-static legge in sola lettura.

## Struttura

```
.
├── docker-compose.yml
├── .env                          # Credenziali (gitignored)
├── nginx-static/
│   └── default.conf
├── filebrowser/
│   ├── .filebrowser.json         # Configurazione
│   └── filebrowser.db            # Database (gitignored)
└── static-files/                 # File da servire (gitignored)
    └── APK/
        └── index.html            # Pagina QR code download APK
```

## Setup iniziale

### 1. Crea la struttura cartelle

```bash
mkdir -p nginx-static static-files filebrowser
```

### 2. Crea la configurazione FileBrowser

```bash
cat > filebrowser/.filebrowser.json << 'EOF'
{
  "port": 80,
  "baseURL": "",
  "address": "",
  "log": "stdout",
  "database": "/database/filebrowser.db",
  "root": "/srv",
  "maxUploadSize": 500
}
EOF
```

### 3. Inizializza il database FileBrowser

> ⚠️ Il db deve essere un **file**, non una directory. Usare sempre `install` per crearlo.

```bash
install -m 644 /dev/null filebrowser/filebrowser.db

docker run --rm \
  -v $(pwd)/filebrowser/filebrowser.db:/database/filebrowser.db \
  -v $(pwd)/filebrowser/.filebrowser.json:/.filebrowser.json \
  filebrowser/filebrowser config init

docker run --rm \
  -v $(pwd)/filebrowser/filebrowser.db:/database/filebrowser.db \
  -v $(pwd)/filebrowser/.filebrowser.json:/.filebrowser.json \
  filebrowser/filebrowser users add admin LA_TUA_PASSWORD --perm.admin
```

### 4. Configura il .env

```bash
nano .env
```

Variabili da compilare:

```properties
PANGOLIN_ENDPOINT=https://app.pangolin.net
NEWT_ID=                      # site nginx-static su Pangolin
NEWT_SECRET=
NEWT_ID_FILEBROWSER=          # site filebrowser su Pangolin
NEWT_SECRET_FILEBROWSER=
NGINX_PORT=80
NGINX_AUTOINDEX=on
STATIC_FILES_PATH=./static-files
NGINX_CONF_PATH=./nginx-static
LOG_MAX_SIZE_NEWT=10m
LOG_MAX_FILES_NEWT=3
LOG_MAX_SIZE_NGINX=20m
LOG_MAX_FILES_NGINX=5
```

### 5. Permessi cartella static-files

```bash
sudo chown -R 1000:1000 static-files/
sudo chmod -R 755 static-files/
```

### 6. Avvia lo stack

```bash
docker compose up -d
```

### 7. Verifica

```bash
docker compose ps
docker logs nginx-static --tail 20
docker logs filebrowser --tail 20
```

## Sicurezza

| Feature | Stato |
|---|---|
| Porte esposte su internet | ✅ Nessuna (tunnel Pangolin) |
| TLS/HTTPS | ✅ Pangolin (Let's Encrypt) |
| Security Headers | ✅ nginx |
| Accesso file nascosti (.env, .git…) | ✅ Bloccato (403) |
| Directory listing | ✅ Abilitato (autoindex on) |
| Volume nginx-static | ✅ Read-only |
| Volume filebrowser | ✅ Scrivibile solo da filebrowser |
| Reti Docker isolate | ✅ static_tunnel_net / filebrowser_net |
| Log rotation | ✅ json-file con limiti |
| Restart automatico | ✅ always |
| Autenticazione upload | ✅ FileBrowser login |

## Comandi utili

```bash
# Stato container
docker compose ps

# Log in tempo reale
docker logs -f nginx-static
docker logs -f newt-static
docker logs -f filebrowser
docker logs -f newt-filebrowser

# Ricarica nginx senza downtime
docker exec nginx-static nginx -s reload

# Fix permessi dopo copia manuale di file
sudo chown -R 1000:1000 static-files/

# Update immagini
docker compose pull
docker compose up -d

# Controlla accessi nginx
docker logs nginx-static 2>&1 | grep -E "GET|POST|404|403" | tail -20
```

## Manutenzione

### Reset password FileBrowser

```bash
docker run --rm \
  -v $(pwd)/filebrowser/filebrowser.db:/database/filebrowser.db \
  -v $(pwd)/filebrowser/.filebrowser.json:/.filebrowser.json \
  filebrowser/filebrowser users update admin --password NUOVA_PASSWORD
```

### Aggiornare nginx

```bash
docker compose pull nginx-static
docker compose up -d nginx-static
```

### Rinnovare certificato
Gestito automaticamente da Pangolin.

### Pagina APK con QR code
Il file `static-files/APK/index.html` legge automaticamente le sottocartelle via autoindex nginx e genera una card con QR code per ogni APK trovato. Disponibile su `https://fileserver.helaglobe.com/APK/`.

## Troubleshooting

### `filebrowser.db` o `.filebrowser.json` creati come directory
Docker crea una directory al posto del file se non esiste sull'host prima del `docker compose up`.

```bash
docker compose down
sudo rm -rf filebrowser/filebrowser.db filebrowser/.filebrowser.json
install -m 644 /dev/null filebrowser/filebrowser.db
# ricreare .filebrowser.json come da setup, poi:
docker compose up -d
```

### 403 su cartelle o file statici
Problema di permessi sul volume. Fix:

```bash
sudo chown -R 1000:1000 static-files/
sudo chmod -R 755 static-files/
docker exec nginx-static nginx -s reload
```

### Newt — `connection refused` su nginx
Verificare che nginx-static sia Up e che i due container siano sulla stessa rete:

```bash
docker compose ps
docker inspect newt-static --format '{{json .NetworkSettings.Networks}}'
docker inspect nginx-static --format '{{json .NetworkSettings.Networks}}'
```

## Note

- **Non committare** `.env`, `filebrowser/filebrowser.db`, `static-files/`
- Creare **due site separati** su Pangolin: uno per nginx-static, uno per filebrowser
- Le reti `static_tunnel_net` e `filebrowser_net` sono completamente isolate
- Questo stack è indipendente dallo stack Odoo — si avvia, aggiorna e spegne separatamente