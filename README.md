# Projet : esp32-airsens 

Version 2

**esp32-airsens** est un projet qui vise à créer des capteurs intelligents et connectés pour la domotique. Ces capteurs utilisent le processeur ESP32 et le langage Micropython pour mesurer différents paramètres environnementaux et les envoyer à une centrale via un réseau sans fil. La centrale est basée sur un système Liligo TTGO qui reçoit les données des capteurs, les publie sur un broker MQTT et affiche les informations pertinentes sur un écran, notamment la température, le taux d’humidité, la pression atmosphérique et le niveau de charge de batterie des capteurs.

La version 2 apporte un nouveau concept pour la structure des données et n'est plus compatible avec la version 1

## Schéma de principe
https://github.com/JOM52/esp32-airsens-tld/blob/main/doc/Sch%C3%A9ma%20de%20principe.jpg

## Elément capteur:

Le hardware de la version 06_tld étant suffisamment abouti, il est repris tel quel.

Afin de minimiser la consommation, cette version n'inclus pour le hardware que le strict minimum pour assurer le fonctionnement du capteur. La communication avec un PC pour la configuration est faite à l'aide d'une interface FTDI (USB to UART) connectée à la demande et les batteries sont rechargées hors du système. 

Dans l'exemple ci-après, le capteur est un BME 280 qui permet la mesure de la température, du taux d'humidité et de la pression atmosphérique.
https://github.com/JOM52/esp32-airsens-tld/blob/main/schema/Airsens%20v06_tld.jpg. 

Des capteurs de type hdc1080 ne permettant que la mesure de la température  et de l'humidité relative peuvent aussi être utilisés

### Hardware

Le hardware ESP32 est prévu pour pouvoir utiliser différents types de capteurs I2C. Actuellement les capteurs BME280, BME680 et HDV1080 sont implémentés.

### Software

Chaque type de capteur a son propre driver. Dans la version 2 le type de capteur utilisé est défini dans le fichier de configuration et il est possible d'avoir plusieurs capteurs sur le même senseur (attention à l'augmentation de la consommation qui va réduire la durée de vie de la batterie).

Comme il s'agit de grandeur "climatiques ou environnementales" une nouvelle mesure chaque 5 minutes est adéquate. Ainsi on peur espérer une durée de vie d'une batterie Li-Ion d'environ 2Ah (type 18650)  d'environ une année.

#### Fichier de configuration du capteur

Afin de faciliter la configuration d'un capteur, un fichier de configuration est lu à chaque démarrage. Ce fichier n'a pas besoin d'une structure particulière, il utilise simplement la syntaxe Micropython pour assigner des valeurs à des variables. De cette façon, les modifications de paramètres se font facilement, dans un seul fichier accessible et sans avoir à chercher et modifier les constantes dans tout le code. L'indication des sections est facultative mais elle aide à comprendre le rôle de chaque paramètre.

```
# acquisitions
SENSOR_LOCATION = 'domo_2' 
T_DEEPSLEEP_MS = 15000
# sensor
SENSORS = {
    'hdc1080':['temp', 'hum'],
    'bme280':['temp', 'hum', 'pres'],
    'bme680':['temp', 'hum', 'pres', 'gas', 'alt']
    }
# power supply
ON_BATTERY = True
UBAT_100 = 4.2
UBAT_0 = 3.1
# I2C hardware config
BME_SDA_PIN = 21
BME_SCL_PIN = 22
# analog voltage measurement
R1 = 977000 # first divider bridge resistor
R2 = 312000 # second divider bridge resistor
DIV = R2 / (R1 + R2)  
ADC1_PIN = 35 # Measure of analog voltage (ex: battery voltage following)
#averaging of measurements
AVERAGING = 1
AVERAGING_BAT = 1
AVERAGING_BME = 1
# ESP-now
PROXY_MAC_ADRESS = b'<a\x05\rg\xcc'
# PROXY_MAC_ADRESS = b'<a\x05\x0c\xe7('
```

### Messages
Les valeurs mesurées pour chaque grandeur sont regroupées et transmises au broker dans un seul message. Le format de principe du message est le suivant:

`msg = (topic, location, grandeur 1, grandeurs 2, .... , granduer n, bat, rssi)`

Les grandeurs mesurées sont dépendantes des possibilités du capteur par exemple:

- HDC1080 : température, humidité
- BME280  : température, humidité, pression
- BME680  : température, humidité, pression, qualité de l'air

Les valeurs transmises à MQTT sont fonction de la configuration définie dans le fichier conf du capteur.

#### Principe du logiciel capteur

Pour économiser l'énergie consommée, le capteur est placé en ***deepsleep*** entre chaque mesure. Le temps de sommeil est défini dans le fichier de configuration en principe 5 minutes. Lors d'un éveil, le capteur lit le fichier de configuration, initialise les constantes, charge les librairies nécessaire, initialise le capteur, fait la mesure, mesure l'état de la batterie, transmets les valeurs à la centrale puis se met en ***deepsleep***

## Centrale

### Software

La centrale est le cœur du système de domotique. Elle a plusieurs fonctions :

- Elle reçoit et traite les messages valides envoyés par les capteurs sans fil.
- Elle adapte les messages au format de l'application de domotique choisie (domoticz, jeedom, ...).
- Elle transmet les valeurs de mesure à MQTT, un protocole de communication qui permet de diffuser les informations aux abonnés.
- Elle affiche différentes informations sur un écran tactile, comme les mesures et l'état des batteries des capteurs.

Les messages reçus ne sont pas confirmés au capteur. Cela signifie que si la centrale n'est pas disponible au moment où le capteur envoie un nouveau message, celui-ci est perdu. Cela ne pose pas de problème pour les mesures de grandeurs physiques qui varient peu dans le temps.

La centrale accepte tous les messages qui ont une structure correcte et qui proviennent d'un capteur dont elle connaît l'adresse MAC. Si le capteur est déjà enregistré dans le fichier de configuration et dans la mémoire programme, les nouvelles valeurs remplacent les anciennes. Sinon, le capteur est automatiquement reconnu et ajouté au fichier de configuration et à la mémoire programme.

La centrale dispose de deux boutons sur la version matérielle des Lilygo-ttgo qui permettent de sélectionner les différents affichages programmés. La sélection d'une option se fait en restant appuyé sur le bouton pendant 2 secondes.

Pour transmettre les mesures à MQTT, la centrale se connecte au réseau WIFI. Les valeurs de mesure sont formatées selon les besoins de l'application de domotique choisie puis envoyées au broker MQTT qui les publie aux abonnés.

### Fichier de configuration de la centrale

Le fichier de configuration est un fichier standard Micropython. 

```
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
file: airsens_central_v2.py 
author: jom52
email: jom52.dev@gmail.com
github: https://github.com/JOM52/esp32-airsens-v2

v0.4.0 : 05.02.2023 --> first prototype
v0.4.1 : 05.03.2023 --> small changes Venezia
v2.0.0 : 11.04.2023 --> adapté pour la version 2
"""
from ubinascii import hexlify
from machine import unique_id

#SYSTEM
WAIT_TIME_ON_RESET = 300 # seconds to wait before the machine reset in case of error

# MQTT
BROKER_IP = '192.168.1.108'
TOPIC = 'airsens_domoticz'
BROKER_CLIENT_ID = hexlify(unique_id())

# TTGO
BUTTON_MODE_PIN = 35
BUTTON_PAGE_PIN = 0
DEFAULT_MODE = 0 # Mode auto
DEFAULT_ROW_ON_SCREEN = 5 # given by the display hardware and the font size
CHOICE_TIMER_MS = 1000 # milli seconds
REFRESH_SCREEN_TIMER_MS = 10000 # mode auto: display next location each ... milli seconds
BUTTON_DEBOUNCE_TIMER_MS = 10 # milli seconds

# WIFI
WIFI_WAN = 'jmb-home'
WIFI_PW = 'lu-mba01'

# BATTERY
BAT_MAX = 4.2 # 100%
BAT_MIN = 3.2 # 0%
BAT_OK = 3.4 # si ubat plus grand -> ok
BAT_LOW = 3.3 # si ubat plus petit -> alarm
BAT_PENTE = (100-0)/(BAT_MAX-BAT_MIN)
BAT_OFFSET = 100 - BAT_PENTE * BAT_MAX
print('% = ' + str(BAT_PENTE) + ' + ' + str(BAT_OFFSET))
```

#### Code de la centrale évolution

*Dans une prochaine version il est prévu de modifier le programme pour:*

- *introduire un processus d'identification et d'enregistrement des capteurs par la centrale. Ainsi seuls des capteurs reconnus, validés et enregistrés seront autorisés à transmettre des mesures à la centrale.*
- *de crypter les message pour empêcher que les messages soient lus par des entités non autorisées.*

### Hardware

La centrale utilise un circuit Lilygo-ttgo qui comprend le processeur, une interface USB, un affichage de 135x240 pixels et deux boutons. Dans la version actuelle, ce circuit est utilisé tel quel. Les boutons sont utilisés comme suit:

- bouton 1 --> next **Mode**

- bouton 2 --> next **Page**

  le choix se fait lorsque aucun bouton n'est pressé pendant 1 secondes

## MQTT mosquitto

Le logiciel MQTT fonctionne comme un éditeur (broker). Il est à l'écoute des auteur (publishers) qui publient des données et les diffuse auprès des abonnés (subscribers).

MQTT est installé comme un service sur un Raspberry PI. Il reçoit les informations depuis les auteurs qui publient les données (les capteurs) et les rediffuse vers tous les abonnés sans aucun contrôle de validité. Son rôle se limite à faire transiter les données depuis les auteurs vers les abonnés. 

Pour recevoir les données depuis le broker il suffit souscrire au broker sur le TOPIC concerné.

*description de l'installation et de la configuration de MQTT à placer ici* 

## Programme python airsens_mqtt

C'est un programme Python qui s'abonne à des topics MQTT pour les capteurs dont les données doivent être enregistrées dans une base de données. Ce programme est installé comme un service sur le raspberry PI. 

Il consulte le fichier de configuration de la centrale pour connaitre les mac-adress des capteurs reconnus par la centrale.

Il reçoit les données depuis MQTT, vérifie que le capteur est reconnu et, si oui, les enregistre dans la base de données airsens sur MariaDB (anciennement MySql) sur le Raspberry PI .

Ce programme se charge également d'alarmer l'utilisateur (par un mail, un WhatsApp ou autre) lorsque l'accumulateur d'un capteur doit être rechargé. 

*description de l'installation d'un service sur RPI à placer ici*

## Programme de domotique

Les mesures des capteurs peuvent, si nécessaire, être transmis à un logiciel de domotique (domoticz, jeedom, ...) pour être représentés graphiquement ou intervenir dans différentes alarmes. 

Ce thème fera objet d'un futur projet.

## GITHUB

Ce projet est maintenu dans GitHub, avec une licence GPL. Il est accessible par le lien 
https://github.com/JOM52/esp32-airsens-tld





Todi, Venise et Pravidondaz février-mars 2023







