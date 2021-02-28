# Projet TMC M2 CRYPTIS

Projet Master 2 CRYPTIS Infos, Terminaux Mobiles Communicants : Mongoose, ESP8266, ATECC508, MQTT, Wi-Fi et Raspberry Pi.

Enoncé du projet sur [Le site de M. Bonnefoi](https://p-fb.net/fileadmin/TMC/2020_2021/Projet_TMC_2020_2021.pdf).

Une grande partie des commandes proviennent du [site de M. Bonnefoi](https://p-fb.net/master-2/tmc.html).

Présentation du projet sur [youtube](https://www.youtube.com/watch?v=iz5PhH7-MNk).

## But du projet

Le but du projet est de créer un réseau de capteurs [ESP8266](https://www.esp8266.com/) connectés par Wi-Fi. La connexion doit respecter le protocole TLS en utilisant des certificats créé grâce à des circuits [ATECC508](https://www.microchip.com/wwwproducts/en/atecc508a) (en courbe elliptiques). Le transfert utilise le protocole MQTT. Ce dernier est proposé par un serveur nommé [Mosquitto](https://mosquitto.org/) tournant sur un [Raspberry Pi 3](https://www.raspberrypi.org/).

## Installation de l'OS Raspberry Pi

L'installation de l'OS Raspberry Pi se fait grâce à l'outils [Raspbery Pi Imager](https://www.raspberrypi.org/software/). C'est un outil disponible sur le site officiel de Raspberry Pi permettant de flasher des cartes mémoires et installer des systèmes d'exploitations.

L'OS utilisé est [Raspberry Pi 64-bit](https://www.raspberrypi.org/forums/viewtopic.php?t=275370).

### Information générale sur l'OS

Le mot de passe par défaut du Raspberry Pi est `raspberry`. Le service SSH est par défaut désactiver. Il est donc nécessaire de connecter le Raspberry Pi à un écran, utiliser un clavier afin de pouvoir activer ce dernier.

Le Raspberry Pi peut potentiellement disposer d'un clavier virtuel pré-installé. Il est disponible dans la section `Preferences` dans le menu déroulant en haut à gauche.

Il est possible également de connecter un clavier par câble USB. 

Installer le clavier virtuel :

```bash
sudo apt-get install matchbox
```

## Activation du SSH sur le Raspberry Pi

L'activation du SSH peut se faire de trois façon différente.

* Utiisation de l'interface graphique

C'est une solution pour ceux qui n'ont pas de clavier, ni de clavier virtuel pré-installé. Il est possible d'activer le SSH grâce à l'interface graphique. Aller sur le `menu`, ensuite `preferences`, `Raspberry pi configuration`. Activer le SSH sur la partie `interfaces` du programme lancé.

* Utiliser `raspi-config`

```bash
sudo raspi-config
```

Selectionner `interfacing options` (déplacement avec les flèches du clavier) et activer le SSH.

* Utiliser `systemctl`

```bash
sudo systemctl enable ssh
sudo systemctl start ssh
sudo systemctl status ssh # Verifier le status actif du protocole
```

## Connexion SSH

Il est conseiller de connecter le Raspberry Pi et l'ordinateur à un Switch. Ce dernier s'occupera d'attribuer les adresses IP, configurer les passerelles etc. Il faut uniquement s'y connecter. Rappel : Le mot de passe est `raspberry`.

```bash
ssh pi@169.254.224.106
```

L'adresse `169.254.224.106` concerne évidement notre Switch. Afin de s'assurer de l'adresse IP du Raspberry Pi. Il faut utiliser l'adresse de l'interface `eth0`.

```bash
sudo ifconfig
```

Comme la seule interface ethernet est occupé par le SSH. Il est maintenant nécessaire de connecter notre Raspberry Pi à internet (Wi-Fi ou câble USB).

## Connexion Wi-Fi ESP8266

Nous cherchons à utiliser une connexion Wi-Fi pour le transfert de données. Il est nécéssaire d'installer des outils de DNS, DHCP et également un outil permettant de configurer un Hotspot.

```bash
sudo apt-get update
sudo apt-get install hostapd
sudo apt-get install dnsmasq
```

### Configuration dnsmasq

Modifier le fichier sur `/etc/dnsmasq.conf` afin d'attribuer les nouvelles configurations DHCP et DNS.

Ajouter la configuration suivante dans le fichier.

```bash
nano /etc/dnsmasq.conf
interface=wlan0
dhcp-range=192.168.1.2,192.168.1.20,255.255.255.0,24h
address=/mqtt.com/192.168.1.1
```

### Configuration hostapd

Ajouter la configuration Hotspot (SSID, mot de passe, TLS, WPA, etc) dans le fichier `/etc/hostapd/hostapd.conf`.

```bash
interface=wlan0
driver=nl80211
ssid=ayoub2bah
hw_mode=g
channel=7
wmm_enabled=0
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=2
wpa_passphrase=ayoubbahbah
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP
```

Editer le fichier `/etc/default/hostapd`. Ajouter la ligne suivante afin de prendre notre fichier de configuration hostapd.

```bash
DEAMON_CONF="/etc/hostapd/hostapd.conf"
```

Activer le forwarding IPv4 en ajoutant (ou en enlevant le commentaire) dans le fichier `/etc/sysctl.conf`

```bash
net.ipv4.ip_forward=1
```

Restart notre Raspberry Pi afin de reload toutes les configurations.

```bash
sudo reboot
```

## Certificats ECC

Dans cette partie nous allons créer l'ensemble de nos certificats et clés utilisés via l'outils `openssl`. Note : Les certificats sont auto-signés.

Créer un fichier `ECC_CERTIFICATES`, afin de regrouper l'ensemble des certificats.

```bash
mkdir /home/pi/ECC_CERTIFICATES
```

### Certificats pour l'AC

```bash
sudo openssl ecparam -out ecc.ca.key.pem -name prime256v1 -genkey
```

```bash
sudo openssl req -config <(printf "[req]\ndistinguished_name=dn\n[dn]\n[ext]\nbasicConstraints=CA:TRUE") -new -nodes -subj "/C=FR/L=Limoges/O=TMC/OU=IOT/CN=ACTMC" -x509 -days 3650 -extensions ext -sha256 -key ecc.ca.key.pem -text -out ecc.ca.cert.pem
```

### Certificats pour le serveur MQTT (Mosquitto)

```bash
sudo openssl ecparam -out ecc.server.key.pem -name prime256v1 -genkey
```

```bash
sudo openssl req -config <(printf "[req]\ndistinguished_name=dn\n[dn]\n[ext]\nbasicConstraints=CA:FALSE") -new -subj "/C=FR/L=Limoges/O=TMC/OU=IOT/CN=serveur.iot.com" -reqexts ext -sha256 -key ecc.server.key.pem -text -out ecc.server.csr.pem
```

```bash
sudo openssl x509 -req -days 3650 -CA ecc.ca.cert.pem -CAkey ecc.ca.key.pem -CAcreateserial -extfile <(printf "basicConstraints=critical,CA:FALSE\n\nsubjectAltName=DNS:localhost") -in ecc.server.csr.pem -text -out ecc.server.pem
```

### Certificats pour le client (ESP8266)

```bash
sudo openssl ecparam -out ecc.client.key.pem -name prime256v1 -genkey
```

```bash
sudo openssl req -config <(printf
"[req]\ndistinguished_name=dn\n[dn]\n[ext]\nbasicConstraints=CA:FALSE") -new -subj
"/C=FR/L=Limoges/O=TMC/OU=IOT/CN=capteur" -reqexts ext -sha256 -key
ecc.client.key.pem -text -out ecc.client.csr.pem
```

```bash
sudo openssl x509 -req -days 3650 -CA ecc.ca.cert.pem -CAkey ecc.ca.key.pem -CAcreateserial -extfile <(printf "basicConstraints=critical,CA:FALSE") -in ecc.client.csr.pem -text -out ecc.client.pem
```

## Installation du serveur MQTT

Installer le serveur MQTT.

```bash
sudo apt-get install mosquitto
sudo apt-get install mosquitto-clients
```

Enlever l'enregistrement des logs. Mettre en commentaire la ligne suivante dans le fichier `/etc/mosquitto/mosquitto.conf`.

```bash
#log_dest file /var/log/mosquitto/mosquitto.log
```

Activer l'authentification par mot de passe sur le serveur MQTT. Ajouter la configuration suivante sur le fichier `/etc/mosquitto/mosquitto.conf`.

```bash
allow_anonymous false
password_file /etc/mosquitto/mosquitto_passwd
```

Créer un utilisateur sur ce fichier :

```bash
sudo mosquitto_passwd -c /etc/mosquitto/mosquitto_passwd ayoub # Ayoub est le username
```

Configuration TLS et TCP. Créer deux fichier dans `/etc/mosquitto/conf.d`.

```bash
touch /etc/mosquitto/conf.d tls.conf
touch /etc/mosquitto/conf.d tcp.conf
```

Ajouter la configuration suivante dans le fichier `tcp.conf`.

```bash
listener 1883
```

Ajouter la configuration suivante dans le fichier `tls.conf`

```bash
listener 8883
cafile /home/pi/ECC_CERTIFICATES/ecc.ca.cert.pem
certfile /home/pi/ECC_CERTIFICATES/ecc.server.pem
keyfile /home/pi/ECC_CERTIFICATES/ecc.server.key.pem
require_certificate true
```

Lancer les services Mosquitto.

```bash
sudo service mosquitto start
sudo service mosquitto restart
sudo service mosquitto status # Vérifier que le service est actif
```

Tester la configuration TLS, les certificats et le serveur MQTT.

```bash
openssl s_client -connect localhost:8883 -CAfile ecc.ca.cert.pem -cert ecc.client.pem -key ecc.client.key.pem
```

Le test doit comprendre une connexion TLS standard, un handshake valide et un contenu chiffré.

### Lacenement du premier Publish et subscribe en localhost

Lancer la commande du subscribe :

```bash
mosquitto_sub -h localhost -p 8883 -t capteur -u ayoub -P ayoubbahbah --cafile /home/pi/ECC_CERTIFICATES/ecc.ca.cert.pem --cert /home/pi/ECC_CERTIFICATES/ecc.client.pem --key /home/pi/ECC_CERTIFICATES/ecc.client.key.pem -d
```

Le Flag -d permet d'avoir davantage d'informations lors du subsbribe.

Dans un deuxième terminal, nous lonçons nos publish sur ce même topic.

```bash
mosquitto_pub -h localhost -p 8883 -t capteur -m 'Bonjour' -u ayoub -P ayoubbahbah --cafile /home/pi/ECC_CERTIFICATES/ecc.ca.cert.pem --cert /home/pi/ECC_CERTIFICATES/ecc.client.pem --key /home/pi/ECC_CERTIFICATES/ecc.client.key.pem -d
```

Le message `Bonjour` doit s'afficher sur le terminal ayant le `mosquitto_sub` lancé.

## Installation du Mongoose-OS

Mongoose-os n'est pas disponible sur apt-get. Il est en PPA, autrement dit, il faut l'ajouter manuellement ensuite update notre apt-get.

Pour ce faire, il faut suivre les étapes suivantes :

Ajouter le [PPA mongoose-os/mos](https://launchpad.net/~mongoose-os/+archive/ubuntu/mos) dans le répértoire apt-get.

```bash
sudo add-apt-repository ppa:mongoose-os/mos
```

Si cette opération est échante, il faut ajouter manuellement les liens suivants dans `/etc/apt/source.list`.

```bash
deb http://ppa.launchpad.net/mongoose-os/mos/ubuntu bionic main 
deb-src http://ppa.launchpad.net/mongoose-os/mos/ubuntu bionic main 
```

Ensuite, update et installer l'outil `mos`.

```bash
sudo apt-get update
sudo apt-get install mos
mos --help
mos
```

## Cloner et modifier le projet Github

Cloner le projet suivant :

```bash
git clone https://github.com/mongoose-os-apps/empty my-app
```

Installer docker avec transfert des droits d'éxecutions à l'utilisateur.

```bash
sudo apt install docker.io
sudo groupadd docker
sudo usermod -aG docker $USER
```

Copier nos certificats vers le dossier `fs` dans le projet cloner.

```bash
sudo cp /home/pi/ECC_CERTIFICATES/ecc.* /home/pi/mongoose-os/my-app/fs
```

Une fois les outils installer, nous procédons à la modification du fichier `mos.yml` dans le dossier cloner `my-app`. Le contenu du fichier est le suivant :

```bash
author: mongoose-os
description: A Mongoose OS app skeleton
version: 1.0
libs_version: ${mos.version}
modules_version: ${mos.version}
mongoose_os_version: ${mos.version}
# Optional. List of tags for online search.
tags:
- c
# List of files / directories with C sources. No slashes at the end of dir names.
sources:
- src
# List of dirs. Files from these dirs will be copied to the device filesystem
filesystem:
- fs
config_schema:
- ["debug.level", 3]
- ["sys.atca.enable", "b", true, {title: "Activation du composant ATEC508"}]
- ["i2c.enable", "b", true, {title: "Enable I2C"}]
- ["sys.atca.i2c_addr", "i", 0x60, {title: "I2C address of the chip"}]
- ["mqtt.enable", "b", true, {title: "Activation du service MQTT"}]
- ["mqtt.server", "s", "mqtt.com:8883", {title: "Adresse du serveur MQTT à joindre"}]
- ["mqtt.pub", "s", "capteur", {title: "Le Topic "}]
- ["mqtt.user", "s", "ayoub", {title: "Utilisateur pour acceder au serveur MQTT"}]
- ["mqtt.pass", "s", "ayoubbahbah", {title: "Mot de passe du serveur MQTT"}]
- ["mqtt.ssl_ca_cert", "s", "ecc.ca.cert.pem", {title: "Le certificat AC pour verifier le   
certificat du serveur"}]
- ["mqtt.ssl_cert", "s", "ecc.client.pem", {title: "Le certificat du client"}]
- ["mqtt.ssl_key", "ATCA:0"]
cdefs:
MG_ENABLE_MQTT: 1
# List of libraries used by this app, in order of initialisation
libs:
- origin: https://github.com/mongoose-os-libs/ca-bundle
- origin: https://github.com/mongoose-os-libs/rpc-service-config
- origin: https://github.com/mongoose-os-libs/rpc-service-atca
- origin: https://github.com/mongoose-os-libs/rpc-service-fs
- origin: https://github.com/mongoose-os-libs/rpc-mqtt
- origin: https://github.com/mongoose-os-libs/rpc-uart
- origin: https://github.com/mongoose-os-libs/wifi
# Used by the mos tool to catch mos binaries incompatible with this file format
manifest_version: 2017-05-18
```

Par la suite, il faut modifier le contenu du fichier `src/main.c` dans le projet cloner. Le contenu est le suivant :

```bash
#include <stdio.h>
#include "mgos.h"
#include "mgos_mqtt.h"
static void my_timer_cb(void *arg) {
char *message = "Hello i am esp8266 !";
mgos_mqtt_pub("capteur", message, strlen(message), 1, 0);
(void) arg;
}
enum mgos_app_init_result mgos_app_init(void) {
mgos_set_timer(2000, MGOS_TIMER_REPEAT, my_timer_cb, NULL);
return MGOS_APP_INIT_SUCCESS;
}
```

Ceci permettra à notre projet de fair les publish sur le bon topic.

Par la suite, il faut build le projet et flasher l'ESP8266.

```bash
sudo mos build --local --arch esp8266
sudo mos flash
```

Configurer notre hotspot sur l'ESP8266  avec `ayoub2bah` le SSID du Wi-Fi et `ayoubbahbah` le mot de passe :

```bash
sudo mos wifi ayoub2bah ayoubbahbah
```

Installer la clé privée sur l'ATECC508.

```bash
sudo openssl rand -hex 32 > slot4.key
sudo mos -X atca-set-key 4 slot4.key --dry-run=false
sudo mos -X atca-set-key 0 ecc.client.key.pem --write-key=slot4.key --dry-run=false
```

Lancer la console mos :

```bash
mos console
```

## Dispositif final

![Photo du dispositif final](https://github.com/larhlimiHamza/tmc-cryptis/blob/main/90f35096-f4e9-4936-8092-56d4825e3ecf.jpg)

## Auteur

[LARHLIMI Hamza](https://github.com/larhlimiHamza)

[BAHBAH Ayoub](https://github.com/BAHBAH-Ayoub)

EL-ASRI Loubna
