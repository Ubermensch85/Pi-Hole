# Guida: Setup Pi-hole + Cloudflared + Tailscale su Debian

**Autore:** Configurazione server DNS privacy-focused con ad-blocking  
**Data:** Febbraio 2026  
**Sistema:** Debian/Ubuntu Server

---

## Indice

1. [Prerequisiti](#prerequisiti)
2. [Installazione Cloudflared](#1-installazione-cloudflared)
3. [Installazione Pi-hole](#2-installazione-pi-hole)
4. [Configurazione dipendenze systemd](#3-configurazione-dipendenze-systemd)
5. [Installazione Tailscale](#4-installazione-tailscale)
6. [Configurazione Tailscale Admin](#5-configurazione-tailscale-admin-global-nameserver)
7. [Ottimizzazione UDP GRO](#6-ottimizzazione-udp-gro-opzionale-per-exit-node)
8. [Test e verifica](#7-test-finale)
9. [Architettura finale](#architettura-finale)
10. [Backup configurazione](#backup-configurazione-opzionale)

---

## Prerequisiti

- Debian/Ubuntu server con accesso root
- IP statico configurato sulla LAN
- Porta 53 libera (nessun servizio DNS attivo)
- Connessione internet stabile

---

## 1. Installazione Cloudflared

Cloudflared fornisce DNS-over-HTTPS (DoH) verso Quad9, crittografando le query DNS.

```bash
# Aggiungi repository Cloudflare
sudo mkdir -p --mode=0755 /usr/share/keyrings
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null
echo "deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/cloudflared.list

# Installa cloudflared
sudo apt update
sudo apt install cloudflared -y

# Crea configurazione Quad9 DoH
sudo mkdir -p /etc/cloudflared
sudo tee /etc/cloudflared/config.yml > /dev/null <<EOF
proxy-dns: true
proxy-dns-port: 5053
proxy-dns-upstream:
  - https://9.9.9.9/dns-query
  - https://149.112.112.112/dns-query
EOF

# Crea servizio systemd
sudo tee /etc/systemd/system/cloudflared.service > /dev/null <<EOF
[Unit]
Description=Cloudflare DNS over HTTPS proxy
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/bin/cloudflared proxy-dns --config /etc/cloudflared/config.yml
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
EOF

# Abilita e avvia
sudo systemctl daemon-reload
sudo systemctl enable --now cloudflared

# Verifica funzionamento
sudo systemctl status cloudflared
sudo ss -tulpn | grep 5053
```

**Output atteso:**
```
udp   UNCONN 0      0      127.0.0.1:5053       0.0.0.0:*    users:(("cloudflared",pid=XXXX,fd=7))
tcp   LISTEN 0      4096   127.0.0.1:5053       0.0.0.0:*    users:(("cloudflared",pid=XXXX,fd=8))
```

---

## 2. Installazione Pi-hole

Pi-hole fornisce ad-blocking a livello DNS per tutta la rete.

```bash
# Installa Pi-hole
curl -sSL https://install.pi-hole.net | bash
```

### Configurazione durante l'installazione:

- **Upstream DNS**: Seleziona **Custom** → inserisci `127.0.0.1#5053`
- **Blocklist**: Lascia default per ora (aggiungeremo Hagezi dopo)
- **Web Admin Interface**: **Sì**
- **Web Server**: **Sì** (lighttpd)
- **Query Logging**: **Sì**
- **Privacy Mode**: Scegli in base alle preferenze

### Aggiungi blocklist Hagezi (TIF + Ultimate)

```bash
# Aggiungi le liste Hagezi al database
sqlite3 /etc/pihole/gravity.db <<EOF
INSERT OR REPLACE INTO adlist (address, enabled, comment) VALUES 
('https://raw.githubusercontent.com/hagezi/dns-blocklists/main/domains/tif.txt', 1, 'Hagezi TIF - 656K domini'),
('https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/ultimate.txt', 1, 'Hagezi Ultimate - 2.8K domini ABP');
EOF

# Aggiorna gravity con le nuove liste
pihole -g

# Verifica liste caricate
pihole -q -adlist
```

### Configura interfaccia per accettare query da tutte le reti

```bash
# Permetti query da Tailscale (100.x.x.x) e LAN (192.168.x.x)
sudo pihole -a -i all

# Verifica configurazione
sudo cat /etc/pihole/dnsmasq.conf | grep -E "server=|listen-address"
```

**Output atteso in `/etc/pihole/dnsmasq.conf`:**
```
server=127.0.0.1#5053
listen-address=0.0.0.0
```

---

## 3. Configurazione dipendenze systemd

Garantisce l'ordine corretto di avvio: **cloudflared → pihole-FTL → tailscaled**

```bash
# Pi-hole dipende da cloudflared
sudo mkdir -p /etc/systemd/system/pihole-FTL.service.d/
sudo tee /etc/systemd/system/pihole-FTL.service.d/override.conf > /dev/null <<EOF
[Unit]
After=cloudflared.service
Wants=cloudflared.service
EOF

# Reload systemd
sudo systemctl daemon-reload

# Verifica dipendenze
systemctl show pihole-FTL | grep -E "After=|Wants=" | grep cloudflared
```

**Output atteso:**
```
Wants=network-online.target nss-lookup.target cloudflared.service
After=basic.target network-online.target sysinit.target system.slice cloudflared.service systemd-journald.socket
```

---

## 4. Installazione Tailscale

Tailscale crea una rete privata VPN mesh per accedere al Pi-hole da ovunque.

```bash
# Aggiungi repository Tailscale
curl -fsSL https://tailscale.com/install.sh | sh

# Avvia Tailscale
# IMPORTANTE: Sostituisci 192.168.1.0/24 con la tua subnet LAN
sudo tailscale up --accept-dns=false --advertise-routes=192.168.1.0/24 --advertise-exit-node

# Login tramite browser (segui il link che appare)
```

**Parametri spiegati:**
- `--accept-dns=false`: Non usare DNS gestiti da Tailscale sul server (evita loop)
- `--advertise-routes=192.168.1.0/24`: Espone la LAN locale alla rete Tailscale
- `--advertise-exit-node`: Permette di usare il server come exit node VPN

### Trova IP Tailscale del server

```bash
tailscale ip -4
# Output esempio: 100.83.109.80
```

Salva questo IP, servirà per configurare il Global Nameserver.

### Configura dipendenza systemd per Tailscale

```bash
# Tailscale dipende da Pi-hole
sudo mkdir -p /etc/systemd/system/tailscaled.service.d/
sudo tee /etc/systemd/system/tailscaled.service.d/override.conf > /dev/null <<EOF
[Unit]
After=pihole-FTL.service
Wants=pihole-FTL.service
EOF

# Reload systemd
sudo systemctl daemon-reload

# Verifica dipendenze
systemctl show tailscaled | grep -E "After=|Wants=" | grep pihole
```

**Output atteso:**
```
Wants=network-pre.target pihole-FTL.service
After=... pihole-FTL.service ...
```

---

## 5. Configurazione Tailscale Admin (Global Nameserver)

Configura tutti i dispositivi Tailscale per usare il tuo Pi-hole come DNS.

1. Vai su **https://login.tailscale.com/admin/dns**
2. Sezione **Global nameservers**: 
   - Clicca **Add nameserver**
   - Inserisci l'IP Tailscale del server (es. `100.83.109.80`)
3. Abilita **Override local DNS** (interruttore)
4. Clicca **Save**

Tutti i dispositivi connessi a Tailscale useranno automaticamente il tuo Pi-hole.

---

## 6. Ottimizzazione UDP GRO (opzionale, per exit node)

Migliora throughput UDP quando si usa Tailscale come exit node.

```bash
# Trova interfaccia di rete principale
ip link
# Output esempio: 1: lo: ... 2: eth0: ... 3: enp1s0: ...

# Abilita UDP GRO forwarding (sostituisci enp1s0 con la tua interfaccia)
sudo ethtool -K enp1s0 rx-udp-gro-forwarding on
sudo ethtool -K enp1s0 rx-gro-list off

# Rendi persistente al riavvio
sudo tee /etc/systemd/system/udp-gro-optimize.service > /dev/null <<EOF
[Unit]
Description=Optimize UDP GRO for Tailscale
After=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/sbin/ethtool -K enp1s0 rx-udp-gro-forwarding on
ExecStart=/usr/sbin/ethtool -K enp1s0 rx-gro-list off
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl enable udp-gro-optimize.service
```

---

## 7. Test finale

### Verifica ordine di avvio servizi

```bash
# Riavvia il server
sudo reboot

# Dopo riavvio, controlla ordine di avvio
sudo journalctl -b | grep -E "(cloudflared|pihole-FTL|tailscaled)" | grep Started
```

**Output atteso (ordine corretto):**
```
... Started Cloudflare DNS over HTTPS proxy
... Started Pi-hole FTL
... Started Tailscale node agent
```

### Test query DNS locale

```bash
# Test risoluzione DNS
dig @127.0.0.1 google.com

# Output atteso: risposta con IP di google.com
```

### Test blocking Pi-hole

```bash
# Query dominio bloccato (WhatsApp analytics)
dig @127.0.0.1 graph.whatsapp.net

# Output atteso: risposta 0.0.0.0 o ::
```

### Test catena completa (cloudflared → Quad9)

```bash
# Stoppa cloudflared temporaneamente
sudo systemctl stop cloudflared

# Prova una query (dovrebbe fallire o timeout)
dig @127.0.0.1 test.com +time=3

# Riavvia cloudflared
sudo systemctl start cloudflared

# Ora dovrebbe funzionare
dig @127.0.0.1 test.com
```

### Monitora query in tempo reale

```bash
# Apri log live Pi-hole
sudo pihole -t

# Apri un browser e naviga su qualsiasi sito
# Dovresti vedere le query apparire nel log
```

### Test da client Tailscale

Da un dispositivo connesso alla rete Tailscale (iPhone, laptop, ecc.):

```bash
# Query DNS verso il Pi-hole (sostituisci con il tuo IP Tailscale)
nslookup google.com 100.83.109.80

# Oppure apri qualsiasi sito web e verifica nei log Pi-hole
# Dovresti vedere l'IP Tailscale del dispositivo (100.x.x.x) nei log
```

### Verifica web interface Pi-hole

1. Apri browser: `http://IP_SERVER/admin`
2. Login con la password impostata durante installazione
3. Dashboard dovrebbe mostrare:
   - **Queries blocked today**: numero crescente
   - **Blocklist**: ~659K domini (Hagezi TIF + Ultimate)
   - **Clients**: dispositivi LAN + Tailscale attivi

---

## Architettura finale

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT DEVICES                            │
├─────────────────────────────────────────────────────────────────┤
│  • Client Tailscale (100.121.62.121) - iPhone/Laptop remoto     │
│  • Client LAN (192.168.1.116) - PC/Laptop locale                │
│  • Server locale (127.0.0.1) - Bot Telegram, servizi locali     │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
         ┌───────────────────────────────┐
         │     Pi-hole FTL (porta 53)    │
         │   • Ad-blocking (659K domini) │
         │   • Query logging             │
         │   • Caching DNS               │
         └───────────┬───────────────────┘
                     │
                     ▼
         ┌───────────────────────────────┐
         │  cloudflared (127.0.0.1:5053) │
         │   • DNS-over-HTTPS (DoH)      │
         │   • Crittografia query        │
         └───────────┬───────────────────┘
                     │
                     ▼
         ┌───────────────────────────────┐
         │       Quad9 (9.9.9.9)         │
         │   • Privacy-focused           │
         │   • DNSSEC validation         │
         │   • Malware blocking          │
         └───────────────────────────────┘
```

**Ordine di avvio systemd:**
```
cloudflared.service → pihole-FTL.service → tailscaled.service
```

**Blocklist attive:**
- **Hagezi TIF**: 656.537 domini esatti
- **Hagezi Ultimate**: 2.808 domini ABP-style

**Upstream DNS:** Quad9 (9.9.9.9 + 149.112.112.112) via DoH

---

## Backup configurazione (opzionale)

### Backup completo

```bash
# Crea backup di tutti i file di configurazione
sudo tar -czf pihole-backup-$(date +%Y%m%d).tar.gz \
  /etc/pihole/ \
  /etc/cloudflared/ \
  /etc/systemd/system/pihole-FTL.service.d/ \
  /etc/systemd/system/tailscaled.service.d/ \
  /etc/systemd/system/cloudflared.service

# Verifica backup creato
ls -lh pihole-backup-*.tar.gz
```

### Restore su nuova macchina

```bash
# Copia il backup sulla nuova macchina, poi:
sudo tar -xzf pihole-backup-YYYYMMDD.tar.gz -C /
sudo systemctl daemon-reload
sudo systemctl restart cloudflared pihole-FTL tailscaled

# Verifica servizi attivi
sudo systemctl status cloudflared pihole-FTL tailscaled
```

---

## Troubleshooting

### Pi-hole non risponde alle query

```bash
# Verifica stato servizi
sudo systemctl status pihole-FTL cloudflared

# Controlla log Pi-hole
sudo journalctl -u pihole-FTL -n 50

# Verifica configurazione upstream
sudo cat /etc/pihole/dnsmasq.conf | grep server=
# Deve mostrare: server=127.0.0.1#5053
```

### Cloudflared non parte

```bash
# Controlla log
sudo journalctl -u cloudflared -n 50

# Verifica configurazione
sudo cat /etc/cloudflared/config.yml

# Test manuale
sudo cloudflared proxy-dns --port 5053 --upstream https://9.9.9.9/dns-query
```

### Tailscale non usa Pi-hole

```bash
# Verifica configurazione Tailscale
tailscale status

# Controlla se accept-dns è false
tailscale status --json | grep -i dns

# Riconfigura se necessario
sudo tailscale up --accept-dns=false --advertise-routes=192.168.1.0/24 --advertise-exit-node
```

### Permessi negati su /etc/pihole/versions

```bash
# Fix permessi Pi-hole
sudo chown -R pihole:pihole /etc/pihole
sudo chmod 644 /etc/pihole/versions
```

---

## Risorse utili

- **Pi-hole Documentation**: https://docs.pi-hole.net/
- **Cloudflared Documentation**: https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/
- **Tailscale Documentation**: https://tailscale.com/kb/
- **Hagezi DNS Blocklists**: https://github.com/hagezi/dns-blocklists
- **Quad9 DNS**: https://www.quad9.net/

---

## Note finali

Questo setup fornisce:
- ✅ **Privacy**: Query DNS crittografate con DoH
- ✅ **Ad-blocking**: 659K domini bloccati
- ✅ **Accesso remoto**: Pi-hole disponibile ovunque tramite Tailscale
- ✅ **Affidabilità**: Dipendenze systemd garantiscono ordine corretto
- ✅ **Performance**: Caching DNS locale, bassa latenza

**Manutenzione consigliata:**
- Aggiorna Pi-hole: `pihole -up` (ogni mese)
- Aggiorna blocklist: `pihole -g` (ogni settimana, automatico)
- Aggiorna Tailscale: `sudo apt update && sudo apt upgrade tailscale`
- Monitora log: `sudo pihole -t` per verificare funzionamento

---

**Guida creata:** Febbraio 2026  
**Sistema testato:** Debian 13 (Trixie) con kernel 6.12.63  
**Hardware consigliato:** Mini PC Intel N100 con 16GB RAM (consumo 6-25W)