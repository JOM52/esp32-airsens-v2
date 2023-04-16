# Projet : esp32-airsens 

Version 2

**esp32-airsens** est un projet de domotique qui consiste à créer des capteurs intelligents et connectés pour surveiller l'environnement. Ces capteurs sont basés sur le processeur ESP32 et le langage Micropython. Ils peuvent mesurer différents paramètres tels que la température, l'humidité, la pression atmosphérique et le niveau de charge de batterie. Ils envoient ensuite ces données à une centrale via un réseau sans fil. La centrale est un système Liligo TTGO qui reçoit les données des capteurs, les publie sur un broker MQTT et les affiche sur un écran. L'écran permet de visualiser les informations pertinentes pour chaque capteur.

La version 2 du projet introduit un nouveau concept pour la structure des données qui n'est pas compatible avec la version 1. Ce concept permet de simplifier la communication entre les capteurs et la centrale, ainsi que de gérer plus facilement les différents types de capteurs.

## Schéma de principe
https://github.com/JOM52/esp32-airsens-v2/blob/main/doc/Sch%C3%A9ma%20de%20principe.jpg

## Elément capteur:

La version 1.06_tld du hardware fonctionne bien et ne nécessite pas de modification.

Pour réduire la consommation d'énergie, cette version hard ne contient que l'essentiel pour faire fonctionner le capteur. La configuration se fait par une interface FTDI (USB to UART) qui se branche au besoin et les batteries se rechargent à part.

Dans l'exemple ci-après, le capteur est un BME 280 qui permet la mesure de la température, du taux d'humidité et de la pression atmosphérique.

https://https://github.com/JOM52/esp32-airsens-v2/blob/main/schema/Schema.pdf

On peut aussi utiliser des capteurs hdc1080 qui mesurent seulement la température et l'humidité relative.

### Hardware

Le hardware ESP32 est un microcontrôleur qui intègre des fonctionnalités de Wi-Fi et de Bluetooth, ainsi que des modules de gestion de l'alimentation, des filtres et des amplificateurs. Il peut se connecter à différents types de capteurs I2C, un protocole de communication série synchrone. 

### Software

Ce projet implémente les drivers pour les capteurs de type BME280, BME680 et HDV1080. Ces capteurs permettent de mesurer différentes grandeurs liées au climat ou à l'environnement. Dans la version 2 du projet, on peut choisir le type et le nombre de capteurs à utiliser dans le fichier de configuration. Toutefois, il faut tenir compte de l'impact sur la consommation d'énergie et la durée de vie de la batterie. Avec une batterie Li-Ion de 2Ah (type 18650), on peut, avec un seul capteur,  réaliser une mesure toutes les 5 minutes pendant environ un an.

#### Fichier de configuration du capteur

Le fichier de configuration permet de personnaliser le fonctionnement d'un capteur sans avoir à modifier le code source. Il contient des lignes de code en Micropython qui assignent des valeurs à des variables globales. Ces variables sont ensuite utilisées par le programme principal pour paramétrer les différents éléments du système. Le fichier de configuration n'a pas besoin d'une structure particulière, mais il peut être organisé en sections pour faciliter la lecture et la compréhension.

Le code ci-après permet de configurer un capteur connecté à un ESP32 qui utilise le protocole ESP-now pour envoyer des données à un proxy. Le capteur peut être de type hdc1080, bme280 ou bme680 et mesure la température, l'humidité, la pression, le gaz et l'altitude selon le cas. Le capteur est alimenté par une batterie et entre en mode deepsleep entre chaque acquisition pour économiser de l'énergie. La tension de la batterie est mesurée par un pont diviseur de tension et un convertisseur analogique-numérique. Le code utilise les constantes définies au début du fichier pour paramétrer les différents éléments du système. Par exemple, SENSOR_LOCATION indique le nom du capteur, T_DEEPSLEEP_MS indique la durée du mode deepsleep en millisecondes, SENSORS indique les types de capteurs disponibles et leurs mesures associées, ON_BATTERY indique si le capteur est alimenté par une batterie ou non, etc. Le code utilise également la variable PROXY_MAC_ADRESS pour indiquer l'adresse MAC du proxy auquel il envoie les données via 

Le code ci-dessous montre un exemple de fichier de configuration pour ce système:

```
# acquisitions
SENSOR_LOCATION = 'domo_2' 
T_DEEPSLEEP_MS = 300000

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
AVERAGING_BAT = 1
AVERAGING_BME = 1

# ESP-now
PROXY_MAC_ADRESS = b'<a\x05\rg\xcc'
# PROXY_MAC_ADRESS = b'<a\x05\x0c\xe7('
```



### Messages

Les valeurs mesurées pour chaque grandeur sont regroupées et transmises au broker MQTT dans un seul message. Le format de principe du message est le suivant:

`msg = (topic, location, grandeur 1, grandeurs 2, .... , grandeur n, bat, rssi)`

Les grandeurs mesurées sont dépendantes des possibilités du capteur par exemple:

- HDC1080 : température, humidité
- BME280  : température, humidité, pression
- BME680  : température, humidité, pression, qualité de l'air

Les valeurs transmises à MQTT sont fonction de la configuration définie dans le fichier _conf du capteur.

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







