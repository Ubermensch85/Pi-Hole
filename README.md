Liste attualmente configurate:

HaGeZi's DNS Blocklist Ultimate 

https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/ultimate.txt

HaGeZi's Threat Intelligence Feeds (TIF) 

https://raw.githubusercontent.com/hagezi/dns-blocklists/main/domains/tif.txt

Ora invece sono passato a Pro + TIF medium (meno aggressiva) e ho whitelistato : 

sudo pihole allow --regex '(.|^)online-metrix.net$' '(.|^)demdex.net$'

sudo pihole allow graph.facebook.com log.tailscale.com h-8om29r2u.online-metrix.net eu-aa.online-metrix.net posteitalianespa.sc.omtrdc.net app-analytics-services.com region1.app-analytics-services.com firebaselogging-pa.googleapis.com firebase-settings.crashlytics.com dpm.demdex.net analytics.adjust.com analytics.adjust.net.in analytics.adjust.world fraud-api.dyneti.com firebaseinstallations.googleapis.com i.paypal.com c6.paypal.com gslb-2.demdex.net enelspa.sc.omtrdc.net smetrics.enelenergia.it webanalytics.regione.umbria.it adobedc.demdex.net crashlyticsreports-pa.googleapis.com sentry.io o4507197393469440.ingest.de.sentry.io geolocation.onetrust.com privacyportal-de.onetrust.com



Per una difesa preconfigurata: 

94 140 14 14

94 140 15 15

DoT : family.adguard-dns.com


