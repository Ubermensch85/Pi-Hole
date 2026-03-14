Liste attualmente configurate:

HaGeZi's DNS Blocklist Ultimate 

https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/ultimate.txt

HaGeZi's Threat Intelligence Feeds (TIF) 

https://raw.githubusercontent.com/hagezi/dns-blocklists/main/domains/tif.txt

Ora invece sono passato a Pro + TIF medium (meno aggressiva) e ho whitelistato : 

# ESSENZIALI (banking, utility, antifraud, firebase, tailscale, facebook)
sudo pihole allow graph.facebook.com i.paypal.com c6.paypal.com posteitalianespa.sc.omtrdc.net enelspa.sc.omtrdc.net smetrics.enelenergia.it webanalytics.regione.umbria.it log.tailscale.com firebaseinstallations.googleapis.com firebase-settings.crashlytics.com crashlyticsreports-pa.googleapis.com firebaselogging-pa.googleapis.com h-8om29r2u.online-metrix.net eu-aa.online-metrix.net dpm.demdex.net adobedc.demdex.net gslb-2.demdex.net fraud-api.dyneti.com geolocation.onetrust.com privacyportal-de.onetrust.com sentry.io o4507197393469440.ingest.de.sentry.io

# Regex per copertura completa domini dinamici
sudo pihole allow --regex '(.|^)online-metrix.net$' '(.|^)demdex.net$'

# Ricarica DNS
sudo pihole reloaddns


Per una difesa preconfigurata: 

94 140 14 14

94 140 15 15

DoT : family.adguard-dns.com


