# Configurazione Ubuntu 24.04 LTS - Frontend Moodle

Il repo rispecchia `/etc` del server: ogni file in `etc/...` va copiato nello stesso percorso sul server.

## Infrastruttura

VM dedicata al frontend (Apache + PHP-FPM). **DB e Redis su macchine separate.**

| Risorsa | Valore |
|---|---|
| CPU | Intel Xeon Gold 6252N - 32 vCPU (2 socket x 16 vCPU) |
| NUMA | 2 nodi vNUMA bilanciati: 16 vCPU + ~32 GB ciascuno |
| RAM | 64 GB (62 GiB visibili al sistema) |
| Disco | SSD 3 TB |
| Hypervisor | VMware vCenter, cluster con vMotion, EVC "Intel Haswell" |
| Software | Apache 2.4.58 (MPM event), PHP-FPM 8.3.27 |

### Note VMware

- **EVC Haswell**: la VM vede la CPU come Haswell (`lscpu` -> `Model: 63`), quindi niente AVX-512 e
  mitigazioni kernel piu' pesanti (PTI, IBRS) rispetto a un Cascade Lake "nudo": qualche punto % di CPU.
  **Si lascia cosi'**: nel cluster ci sono host diversi e EVC serve a vMotion/HA. Un EVC per-VM
  limiterebbe gli host su cui la VM puo' migrare o essere riavviata da HA.
- **vNUMA 2x16**: topologia corretta e allineata a vCenter (2 socket x 16). La topologia e' fissata
  all'accensione: gli host su cui la VM puo' migrare devono avere >= 16 core fisici per socket e
  >= 32 GB per nodo NUMA (eventualmente regola DRS VM->Host di tipo "dovrebbe").
- **Memory Reservation = 64 GB** ("Reserve all guest memory"): consigliata, evita ballooning/swap
  lato ESXi visto che la RAM viene usata quasi interamente.

## Budget RAM

| Componente | RAM |
|---|---|
| OS, VMware Tools, monitoraggio | ~2 GB |
| Apache event (16 processi x 64 thread) | ~1 GB |
| OPcache (memoria condivisa, unica per tutti i worker) | 1 GB |
| **PHP-FPM: 256 worker x ~128 MB (stima prudente)** | **~32 GB** |
| Margine picchi (backup, report pesanti) | ~6 GB |
| Page cache filesystem (codice Moodle, moodledata/localcache) | ~20 GB |

Il limite reale sono le **32 vCPU**, non la RAM: con ~400 worker (tetto RAM) i processi si
contenderebbero la CPU peggiorando i tempi di risposta. La RAM non assegnata a FPM non e' sprecata:
Linux la usa come cache dei file.

Misure in produzione (configurazione precedente, carico basso): `used 3.7 GiB`, `available 59 GiB`,
120 worker FPM attivi -> ~20-25 MB per worker a riposo. Il dato da confermare e' il consumo al picco.

---

## Modifiche effettuate

### Apache

**[etc/apache2/mods-available/mpm_event.conf](etc/apache2/mods-available/mpm_event.conf)** (rinominato da `mpm_events.conf`)

Il nome con la "s" non viene collegato da `a2enmod`: Apache usava i default Ubuntu (`MaxRequestWorkers 150`).

| Parametro | Valore | Perche' |
|---|---|---|
| `StartServers` | 4 | 256 thread pronti all'avvio |
| `ServerLimit` / `ThreadsPerChild` / `ThreadLimit` | 16 / 64 / 64 | 16 x 64 = 1024 thread |
| `MaxRequestWorkers` | 1024 | rapporto ~4:1 con i worker FPM: keep-alive, HTTP/2 e statici occupano thread Apache ma non PHP |
| `MinSpareThreads` / `MaxSpareThreads` | 128 / 512 | riserva per i picchi |
| `MaxConnectionsPerChild` | 0 (era 10000) | Apache non esegue PHP, nessun leak da contenere |
| `AsyncRequestWorkerFactor` | 2 | connessioni keep-alive gestite in modo asincrono |

**[etc/apache2/apache2.conf](etc/apache2/apache2.conf)** (estratto delle modifiche)

- `Timeout 600` / `ProxyTimeout 600`: allineati a `request_terminate_timeout` di FPM.
- Deflate: aggiunti `text/javascript` e `image/svg+xml`.

**[etc/apache2/sites-available/000-moodle.conf](etc/apache2/sites-available/000-moodle.conf)**

- **XSendFile** (`XSendFilePath /var/moodledata`): i download (pluginfile, video, SCORM, backup) li
  serve Apache; senza, ogni download occupa un worker PHP per tutto il trasferimento.
  Richiede `$CFG->xsendfile = 'X-Sendfile';` in `config.php`.
- **Monitoraggio**: `/server-status` (Apache, mod_status) e `/fpm-status`, `/fpm-ping` (PHP-FPM)
  accessibili **solo da localhost e dagli IP in `MONITORING_IPS`** (`Define` in testa al file),
  e esclusi dall'access log. Vedi la sezione "Monitoraggio remoto".
- Regole file sensibili ancorate (`^config\.php$`, ...) per non bloccare file legittimi;
  blocco `vendor|node_modules|tests|fixtures` esteso alle sottocartelle.
- Rimosso `X-XSS-Protection` (deprecato, ignorato dai browser).
- `LogLevel warn` (era una sintassi ambigua), `font/woff2` corretto, aggiunti `image/webp` e `text/javascript` agli Expires.
- Commenti corretti: la cartella `public/` esiste da **Moodle 5.1**; il router `r.php` richiede `$CFG->routerconfigured = true;`.

### PHP-FPM

**[etc/php/8.3/fpm/pool.d/www.conf](etc/php/8.3/fpm/pool.d/www.conf)**

| Parametro | Valore | Perche' |
|---|---|---|
| `pm` | `static` (era `dynamic`) | server dedicato: nessun costo di fork nei picchi (FPM in `dynamic` crea max 32 processi/s, a cache fredda), consumo RAM prevedibile |
| `pm.max_children` | 256 (era 300) | 8 x vCPU. Tetto RAM ~400: alzare a 320-384 solo se al picco CPU < 70% e compare "max_children reached" |
| `pm.max_requests` | 500 | riciclo worker contro memory leak |
| `listen.backlog` | 1024 | coda richieste quando tutti i worker sono occupati (<= `somaxconn`) |
| `pm.status_path` / `ping.path` | `/fpm-status` / `/fpm-ping` | monitoraggio |
| `request_terminate_timeout` | 600s | = timeout Apache |
| `access.log` | disattivato | con 256 worker genera molto I/O; riattivare solo per debug |
| `memory_limit` | 512M con **`php_value`** (era `php_admin_value` 1024M) | con `php_admin_value` Moodle non puo' alzare i limiti da codice (`raise_memory_limit`, `core_php_time_limit::raise`) per backup/restore/report |
| `max_execution_time` / `max_input_time` | 600 con `php_value` | come sopra |
| `post_max_size` / `upload_max_filesize` | 512M / 256M | invariati |
| `pm.process_idle_timeout` | rimosso | vale solo con `pm = ondemand` |

**[etc/php/8.3/mods-available/moodle-opcache.ini](etc/php/8.3/mods-available/moodle-opcache.ini)** (nuovo, `; priority=99`)

OPcache configurata in un solo punto. Prima c'era un conflitto: `php.ini` impostava `opcache.jit = tracing`,
ma `opcache.ini` (priority 10, caricato dopo) lo riportava a `off`. File separato perche' `opcache.ini`
e' del pacchetto `php8.3-opcache` ed e' condiviso con la CLI.

| Parametro | Valore | Perche' |
|---|---|---|
| `opcache.memory_consumption` | 1024 | bytecode di Moodle + plugin, condiviso tra tutti i worker |
| `opcache.interned_strings_buffer` | 128 | |
| `opcache.max_accelerated_files` | 65407 | Moodle + plugin ~30-40k file PHP |
| `opcache.save_comments` | 1 | obbligatorio per Moodle |
| `opcache.validate_timestamps` / `revalidate_freq` | 1 / 60 | i file modificati vengono ricaricati entro 60s |
| `opcache.jit` / `jit_buffer_size` | off / 0 | Moodle e' limitato da I/O e DB: guadagno trascurabile |
| `opcache.huge_code_pages` | 0 | richiede hugepages configurate nel kernel |

**[etc/php/8.3/fpm/php.ini](etc/php/8.3/fpm/php.ini)**

- Rimossa la sezione `[opcache]` (ora in `moodle-opcache.ini`).
- `max_execution_time`/`max_input_time` 600 e `post_max_size` 512M: allineati al pool, per non avere
  due valori diversi (il pool ha comunque la precedenza).
- Corretti i commenti su `cgi.fix_pathinfo` (default 1, necessario per gli slasharguments) e sulle sessioni (Redis).

**[etc/php/8.3/cli/php.ini](etc/php/8.3/cli/php.ini)**

- `opcache.enable_cli = 0`: cron e script sono processi brevi, la cache non sopravvive all'esecuzione.
- `error_log = /var/log/php/php8.3-cli.error.log`: separato da FPM (evita conflitti di permessi root/www-data).

**Xdebug**: [mods-available/xdebug.ini](etc/php/8.3/mods-available/xdebug.ini) resta nel repo, ma va
**disabilitato per FPM** (`phpdismod -s fpm xdebug`): anche con `xdebug.mode=off` resta caricato e disattiva il JIT.

**[etc/systemd/system/php8.3-fpm.service.d/override.conf](etc/systemd/system/php8.3-fpm.service.d/override.conf)** (nuovo)

`LimitNOFILE=65536`. Il master FPM tiene ~2 descrittori per worker: con 256 worker sono ~520 fd,
troppo vicino al default soft di 1024 (verificato in produzione: `1024 / 524288`).
Apache non ne ha bisogno: `apache2ctl` imposta gia' `ulimit -n 8192` (via `APACHE_ULIMIT_MAX_FILES`
in `/etc/apache2/envvars`), sufficiente, e un override systemd non avrebbe effetto.

### Sistema

**[etc/sysctl.d/99-moodle.conf](etc/sysctl.d/99-moodle.conf)** (sostituisce i parametri custom di `/etc/sysctl.conf`)

| Parametro | Valore | Perche' |
|---|---|---|
| `vm.overcommit_memory` | **0** (era 1) | 1 e' una raccomandazione per server Redis, che qui non c'e'; con 1 il kernel non rifiuta mai allocazioni e interviene l'OOM killer |
| `vm.swappiness` | 1 | invariato |
| `vm.vfs_cache_pressure` | 50 | invariato: tiene in cache dentry/inode dei molti file Moodle |
| `fs.file-max` | **rimosso** (era 2000000) | il default di Ubuntu 24.04 e' ~9.2e18: impostarlo lo abbassava. Conta il limite per processo |
| `net.core.somaxconn` / `netdev_max_backlog` / `tcp_max_syn_backlog` | 4096 | invariati |
| `net.ipv4.tcp_syncookies` | 1 | invariato |
| `net.ipv4.tcp_fin_timeout` | 15 | invariato |
| `net.ipv4.tcp_keepalive_time` | 1800 | invariato (deve restare sotto l'idle timeout dei firewall verso DB/Redis) |
| `net.ipv4.ip_local_port_range` | **10240 65535** (era 1024) | evita che una connessione in uscita verso DB/Redis occupi una porta di un servizio locale |
| `net.ipv4.tcp_tw_reuse` | 1 | invariato: utile perche' questa VM e' client verso DB/Redis |
| `net.ipv4.tcp_slow_start_after_idle` | **0** (nuovo) | le connessioni keep-alive (HTTP/2, DB, Redis) non ripartono lente dopo una pausa |

Verificati in produzione: `vm.zone_reclaim_mode = 0` e `kernel.numa_balancing = 1` (corretti con 2 nodi NUMA, nessuna modifica).

**[etc/logrotate.d/php-moodle](etc/logrotate.d/php-moodle)** (nuovo): rotazione giornaliera, 14 giorni, dei log in `/var/log/php/`.
La directory `/var/log/php` deve essere di **root** (`0755`), con i soli file di log di `www-data`:
logrotate gira come root e in una directory scrivibile da `www-data` rifiuterebbe di ruotare
("insecure permissions") o sarebbe esposto ad attacchi via symlink.

### Rimossi

- `etc/apache2/sites-available/001-moodledev.conf` (multi-istanza `prod`/`m405`/`m500`: le istanze non girano su questo server).

---

## Applicazione sul server

Dalla cartella del repo sul server (dopo `git pull`):

```bash
# --- Apache ---
sudo rm -f /etc/apache2/mods-available/mpm_events.conf          # vecchio nome, non caricato
sudo install -m 644 etc/apache2/mods-available/mpm_event.conf  /etc/apache2/mods-available/
sudo install -m 644 etc/apache2/sites-available/000-moodle.conf /etc/apache2/sites-available/
# apache2.conf nel repo e' un ESTRATTO: riportare le modifiche a mano in /etc/apache2/apache2.conf
sudo apt install libapache2-mod-xsendfile
sudo a2enmod xsendfile expires headers http2 proxy_fcgi setenvif
sudo apachectl configtest && sudo systemctl reload apache2

# --- PHP-FPM ---
sudo install -m 644 etc/php/8.3/fpm/pool.d/www.conf          /etc/php/8.3/fpm/pool.d/
sudo install -m 644 etc/php/8.3/fpm/php.ini                  /etc/php/8.3/fpm/
sudo install -m 644 etc/php/8.3/cli/php.ini                  /etc/php/8.3/cli/
sudo install -m 644 etc/php/8.3/mods-available/moodle-opcache.ini /etc/php/8.3/mods-available/
sudo phpdismod -s fpm xdebug
sudo phpenmod  -s fpm moodle-opcache        # crea fpm/conf.d/99-moodle-opcache.ini
# directory dei log di root (NON scrivibile da www-data: logrotate gira come root),
# file di log pre-creati di www-data perche' i worker FPM e il cron possano scriverci
sudo install -d -o root -g adm -m 0755 /var/log/php
sudo install -o www-data -g adm -m 0640 /dev/null /var/log/php/php8.3-fpm.error.log
sudo install -o www-data -g adm -m 0640 /dev/null /var/log/php/php8.3-cli.error.log
sudo install -m 644 etc/logrotate.d/php-moodle /etc/logrotate.d/

# limite file aperti (in alternativa: systemctl edit php8.3-fpm, scrivendo SOPRA la riga
# "### Edits below this comment will be discarded")
sudo mkdir -p /etc/systemd/system/php8.3-fpm.service.d
sudo install -m 644 etc/systemd/system/php8.3-fpm.service.d/override.conf /etc/systemd/system/php8.3-fpm.service.d/
sudo systemctl daemon-reload

# restart completo (reload NON applica il nuovo LimitNOFILE). Interrompe per un istante le richieste PHP.
sudo php-fpm8.3 -t && sudo systemctl restart php8.3-fpm

# --- sysctl ---
sudo install -m 644 etc/sysctl.d/99-moodle.conf /etc/sysctl.d/
sudo nano /etc/sysctl.conf      # commentare/rimuovere i parametri custom: e' letto DOPO
                                # (symlink /etc/sysctl.d/99-sysctl.conf) e vincerebbe
sudo sysctl --system
```

**Moodle `config.php`**:

```php
$CFG->xsendfile = 'X-Sendfile';
$CFG->routerconfigured = true;
```

## Verifiche dopo l'applicazione

```bash
# MPM caricato con i valori giusti
apachectl -t -D DUMP_RUN_CFG ; apachectl -M | grep -E "mpm|xsendfile|http2"

# OPcache e JIT effettivi, Xdebug assente
php-fpm8.3 -i | grep -E "opcache\.(enable|memory_consumption|jit) " ; php-fpm8.3 -m | grep -i xdebug

# limite file aperti del master FPM -> atteso 65536 65536
sudo grep "open files" /proc/$(pgrep -o php-fpm8.3)/limits

# sysctl -> atteso 0 | 10240 65535 | 0
sysctl vm.overcommit_memory net.ipv4.ip_local_port_range net.ipv4.tcp_slow_start_after_idle

# worker FPM (atteso 257 = 256 + master) e stato del pool
ps -C php-fpm8.3 --no-headers | wc -l
curl -sk "https://localhost/fpm-status"
```

## Monitoraggio remoto

In testa a [000-moodle.conf](etc/apache2/sites-available/000-moodle.conf) impostare l'IP (o la rete)
del server di monitoraggio; piu' valori separati da spazio:

```apache
Define MONITORING_IPS "10.0.5.20 10.0.6.0/24"
```

Finche' resta il valore di esempio (`192.0.2.10`, rete di documentazione) accede solo localhost.
Le pagine sono solo in HTTPS (il vhost :80 reindirizza tutto).

| URL | Contenuto | Formato per tool |
|---|---|---|
| `/server-status?auto` | Apache: richieste/s, byte/s, thread occupati/liberi, stato connessioni (keep-alive, chiusura) | testo chiave: valore |
| `/server-status` | come sopra + elenco richieste in corso (client, URL, tempi) | HTML |
| `/fpm-status?json` | FPM: processi attivi/idle, coda di attesa, `max children reached`, `max active processes`, `slow requests` | JSON (anche `?openmetrics` per Prometheus) |
| `/fpm-status?full&json` | come sopra + dettaglio di ogni worker (script, durata, memoria) | JSON |
| `/fpm-ping` | risponde `pong` se FPM e' vivo | testo |

Metriche da tenere d'occhio:
- **Apache** `BusyWorkers` vicino a 1024 -> Apache saturo; `ConnsAsyncKeepAlive` alto e' normale.
- **FPM** `listen queue` > 0 in modo stabile o `max children reached` che cresce -> pool saturo.
- **FPM** `slow requests` in crescita -> controllare `/var/log/php/php8.3-fpm.slow.log`.

`/server-status` mostra IP dei client e URL delle richieste in corso: non deve mai essere
raggiungibile da Internet. `mod_status` e' attivo di default su Ubuntu (`a2enmod status`);
il suo `status.conf` di default resta invariato (`ExtendedStatus On`, `ProxyStatus On`).

Prova dal server di monitoraggio:

```bash
curl -s  "https://moodledev.esempio.dev/server-status?auto" | head
curl -s  "https://moodledev.esempio.dev/fpm-status?json"
```

## Verifica dimensionamento (da fare al picco: quiz, inizio lezioni)

```bash
free -h
sudo smem -P php-fpm -c "pid pss" | awk 'NR>1{s+=$2;n++} END{print s/n/1024 " MB medi"}'   # apt install smem
sudo grep -E "max_children|seems busy" /var/log/php8.3-fpm.log* | tail -20
```

- CPU < 70%, RAM disponibile e "max_children reached" nel log -> alzare `pm.max_children` a 320-384.
- CPU ~100% -> **non** alzare: servono piu' vCPU o un secondo frontend. Controllare anche il **CPU Ready**
  della VM in vCenter (deve restare sotto ~5%).

## Comandi diagnostici gia' eseguiti in produzione

| Comando | Risultato | Conclusione |
|---|---|---|
| `numactl -s` / `numactl -H` | 2 nodi: 0-15 / 16-31, ~32 GB ciascuno, distanze 10/20 | vNUMA bilanciato, ok |
| `lscpu` | Xeon Gold 6252N, `Model: 63`, mitigazioni PTI/IBRS | EVC Haswell attivo (confermato in vCenter), si lascia |
| `free -h` | used 3.7 GiB, buff/cache 57 GiB, available 59 GiB | nessun consumo anomalo, RAM libera = page cache |
| `ps -C php-fpm8.3 --no-headers \| wc -l` | 121 | 120 worker + master (config `dynamic` precedente) |
| `grep "open files" /proc/<pid>/limits` | Apache 8192/8192, FPM 1024/524288 | FPM da alzare -> override systemd |
| `sysctl vm.zone_reclaim_mode kernel.numa_balancing` | 0 / 1 | corretti, nessuna modifica |
