# Zendure Solarflow - Daten an lokalen mqtt-Broker senden und in Home-Assistent abfragen

## Vorwort

Seit ca. 1 Jahr habe ich den Zendure Solarflow Hub1200 im Einsatz. Ich suchte nach einer Möglichkeit die Daten in Home-Assistent einzubinden. Dazu nutze ich bis vor kurzer Zeit die Möglichkeit der Abfrage von mqtt-Daten aus der Zendure-Cloud nach der Anleitung von z-master42 (https://github.com/z-master42/solarflow).
Leider musste ich feststellen, dass sich diese Daten auf Grund sehr hoher Latenzen, nur bedingt zur Auswertung im Energy-Dashboard von Home-Assistent eignen.
Dann bin ich auf eine Anleitung von Reinhard Brandstaedter (https://github.com/reinhard-brandstaedter/solarflow-bt-manager) gestoßen, in der die lokale Abfrage der mqtt-Daten von Zendure beschrieben wird.

Nachfolgend erkläre ich meine Vorgehensweise.

Ich weise jedoch darauf hin, dass die Nutzung dieser Beschreibung auf eigene Gefahr erfolgt. Ich hafte nicht für eventuelle Schäden, Funktionsstörungen oder sonstige Dinge die durch die Nutzung dieser Anleitung enstehen.

## Voraussetzungen

  - Home-Assistent
  - PC oder Laptop mit Bluetooth und Python (ich habe einen Windows-Laptop benutzt)
  - lokaler mqtt-Broker mit anonymen Login
  - Hub1200 (SF_PRODUCT_ID=73bkTV) oder Hub2000 (SF_PRODUCT_ID=A8yh63) oder AIO2400 (SF_PRODUCT_ID=yWF7hV)

## Steps

### 1. lokaler mqtt-Broker

Ich empfehle hier EMQX. Dieser mqtt-Broker ist als Addon für Home-Assistent verfügbar, kann aber auch z.B. als LXC auf Proxmox laufen.
Der vielbenutzte Mosquitto broker in Homeassistent funktioniert für dieses Vorhaben nicht, da er keine anonymen Logins zulässt.

EMQX installieren und Weboberfläche des EMQX öffnen.
Im Menü unter "Authentication" eventuelle Einträge löschen.

### 2. Scripte herunterladen, Zendure-Infos abfragen und Scripe anpassen

Link aufrufen und nachfolgende Dateien herunterladen: https://github.com/reinhard-brandstaedter/solarflow-bt-manager/tree/master/src

  - requirements.txt
  - solarflow-bt-manager.py
  - solarflow-topic-mapper.py

Diese 3 Dateien in ein beliebiges Verzeichnis packen.
Windows-Powershell starten und in das Verzeichnis wechseln in dem die 3 Dateien liegen.

Script aufrufen mit folgendem Befehl:
```
$ python3 solarflow-bt-manager.py -i
```
Das Script sollte nun Abfragen ausführen und dann stoppen. Es werden folgende Informationen benötigt.
```
2024-10-10 14:31:12,682:INFO: The SF device ID is: XXXXXXXXXXX
2024-10-10 14:31:12,684:INFO: The SF device SN is: XXXXXXXXXXX
```

Dateien anpassen:

In den Dateien "solarflow-bt-manager.py" und "solarflow-topic-mapper.py" müssen folgende Anpassungen vorgenommen werden.

solarflow-bt-manager.py:
```
WIFI_PWD = os.environ.get('WIFI_PWD','WLAN_PASSWORT')
WIFI_SSID = os.environ.get('WIFI_SSID','WLAN_SSID')
SF_DEVICE_ID = os.environ.get('SF_DEVICE_ID','DEVICE_ID') #siehe Abfrage-Infos Script
SF_PRODUCT_ID = os.environ.get('SF_PRODUCT_ID','PRODUCT_ID') #siehe oben bei Voraussetzungen
```

solarflow-topic-mapper.py:
```
sf_device_id = os.environ.get('SF_DEVICE_ID','DEVICE_ID') **siehe Abfrage-Infos Script**
sf_product_id = os.environ.get('SF_PRODUCT_ID',"PRODUCT_ID") **siehe oben bei Voraussetzungen**
mqtt_user = os.environ.get('MQTT_USER',None)
mqtt_pwd = os.environ.get('MQTT_PWD',None)
mqtt_host = os.environ.get('MQTT_HOST','IP_MQTT-BROKER')
mqtt_port = os.environ.get('MQTT_PORT',1883)
```

### 3. SF-Device von der Zendure-Cloud trennen

**WICHTIG!!!! Zendure-App auf dem Smartphone schließen**
In der Windows Powershell feolgenden Befehl ausführen.
```
$ python3 solarflow-bt-manager.py -d -w WLAN_SSID -b IP_MQTT-BROKER:1883
```

Jetzt sollten im MQTT-Explorer schon ein Topic zu sehen sein. 
```
/73bkTV/<your device id>/#
```
Um die mqtt-Topics für Home-Assistent aufzubereiten folgt der nächste Schritt.

### 4. mqtt-Topics

Hier kann erst einmal der Laptop mit der Powershell genutzt werden. Allerdings muss nachfolgendes Script permanent laufen und dazu müsste der Laptop immer eingeschaltet sein. Da dies ziemlich unvorteilhaft ist, habe ich unter Proxmox einen Linux-Container eingerichtet in dem ich nachfolgendes Script permanent laufen lasse. Es gibt aber auch andere Möglichkeiten z.B. Docker usw.
```
$ python3 solarflow-topic-manager.py
```

Jetzt sollten im MQTT-Explorer unter dem Haupttopic "solarflow-hub"alle Untertopics zu sehen sein.

### 5. Home-Assistent mqtt.yaml

In Home-Assistent in der configuration.yaml folgende Zeite einfügen.
```
mqtt: !include mqtt.yaml
```

mqtt.yaml herunter laden und anpassen. WICHTIG!!! Die Einrückungen unbedingt beibehalten!!!
Nach dem Anpassen im config-Verzeichnis des Homeassistant ablegen.
Homeassistant neu starten!

**Das Zendure-Device ist jetzt nicht mehr in der Zendure-Cloud aber über die Zendure-App per Bluetooth weiterhin konfigurierbar.**

### 6. SF-Device zurück in die Zendure-Cloud bringen, wenn nötig

Dazu wird folgender Befehl benutzt:
```
$ python3 solarflow-bt-manager.py -c -w <WiFi SSID>
```
