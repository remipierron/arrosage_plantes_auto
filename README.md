# Arrosage automatique de plantes avec Raspberry Pi 4

Projet d'arrosage automatique piloté par un Raspberry Pi 4 (2 Go de RAM). Phase 1 : valider le système sur **1 plante**. Phase 2 : passer à **10 plantes** sur le même Raspberry Pi.

## Objectifs

- Mesurer l'humidité du sol, la température et l'humidité de l'air
- Déclencher l'arrosage automatiquement quand le sol est trop sec
- Sécuriser le système (réservoir vide, capteur défaillant, pompe bloquée)
- Extensible à 10 plantes sans changer l'architecture

## Matériel

### Capteurs

| Élément | Référence | Qté (1 → 10) |
|---|---|---|
| Humidité du sol | Capacitive Soil Moisture Sensor v1.2 | 1 → 10 |
| Convertisseur analogique/numérique | ADS1115 (I2C, 16 bits, 4 entrées) | 1 → 3 |
| Température/humidité de l'air | BME280 (I2C) | 1 → 1 |
| Température du sol (optionnel) | DS18B20 étanche (1-Wire) | 1 → 10 |
| Niveau d'eau du réservoir | Interrupteur à flotteur | 1 → 1 |

### Arrosage

| Élément | Référence | Qté |
|---|---|---|
| Pompes | Pompe péristaltique 12 V (ou mini pompe immergée) | 1 → 10 |
| Commande des pompes | Module relais 8 canaux 5 V optocouplé | 1 → 2 |
| Alimentation pompes | Alimentation 12 V 5 A | 1 |
| Alimentation du Pi | Alimentation officielle Raspberry Pi 5,1 V 3 A USB-C | 1 |
| Tuyaux | Silicone 4×6 mm, 5 à 10 m | 1 |
| Réservoir | Bidon ou bac opaque | 1 |
| Diodes de roue libre | 1N4007 (une par pompe) | 1 → 10 |

### Câblage

- Câbles Dupont (mâle/femelle et femelle/femelle)
- Bornes à levier Wago (2 et 5 entrées)
- Plaque de prototypage ou perfboard
- Câble 3 conducteurs pour les capteurs éloignés
- Boîtier étanche pour l'électronique

## Architecture

```
                  ┌─────────────────────────┐
                  │     Raspberry Pi 4      │
                  │                         │
  I2C (SDA/SCL) ──┤  ADS1115 x1..3 (0x48-4B)│◄── capteurs d'humidité du sol
                  │  BME280 (0x76)          │◄── température/humidité air
  1-Wire (GPIO4) ─┤  DS18B20 x N (optionnel)│
  GPIO ───────────┤  Flotteur (entrée)      │◄── niveau du réservoir
  GPIO ───────────┤  Relais 8 canaux x1..2  │──► pompes 12 V
                  └─────────────────────────┘
```

Points clés :

- Le Pi n'a pas d'entrée analogique : les capteurs d'humidité passent par des ADS1115.
- Un seul bus I2C suffit : 3 ADS1115 + 1 BME280 = 4 adresses distinctes, sans conflit.
- Les capteurs sont alimentés en **3,3 V** pour que leur sortie reste compatible avec l'ADC.
- Les pompes sont alimentées par une alimentation 12 V séparée, commutée par les relais.
- Une pompe par plante (plus simple et plus fiable que des électrovannes).

## Plan du projet

### Phase 0 : Préparation

- [ ] Commander le matériel pour 1 plante
- [ ] Installer Raspberry Pi OS Lite, activer SSH, I2C et 1-Wire (`raspi-config`)
- [ ] Vérifier la détection I2C : `i2cdetect -y 1`

### Phase 1 : Prototype sur 1 plante

- [ ] Câbler l'ADS1115 et le capteur d'humidité du sol
- [ ] Câbler le BME280
- [ ] Lire et afficher les valeurs en console
- [ ] Calibrer le capteur d'humidité (valeur à sec dans l'air, valeur à saturation dans l'eau)
- [ ] Câbler un relais, une pompe, une diode de roue libre et son alimentation 12 V
- [ ] Tester la pompe manuellement et mesurer le débit (ml/s)
- [ ] Câbler le flotteur du réservoir
- [ ] Écrire la logique d'arrosage : seuil bas, durée d'arrosage, délai minimum entre deux arrosages
- [ ] Ajouter les sécurités (voir plus bas)
- [ ] Faire tourner le système 1 à 2 semaines et ajuster les seuils

### Phase 2 : Journalisation et supervision

- [ ] Enregistrer les mesures (SQLite ou CSV) avec horodatage
- [ ] Enregistrer chaque arrosage (plante, durée, quantité)
- [ ] Lancer le programme comme service `systemd` (démarrage automatique, redémarrage en cas de plantage)
- [ ] Interface de consultation : page web légère (Flask) ou Grafana, au choix
- [ ] Notifications (réservoir vide, capteur hors plage) via Telegram, ntfy ou e-mail

### Phase 3 : Passage à 10 plantes

- [ ] Ajouter 2 ADS1115 (adresses 0x49 et 0x4A)
- [ ] Ajouter un second module relais pour dépasser 8 canaux
- [ ] Faire passer la configuration en fichier YAML : une entrée par plante
- [ ] Étiqueter capteurs, pompes et tuyaux
- [ ] Passer à un câblage soudé (perfboard) dans un boîtier
- [ ] Vérifier la consommation cumulée des pompes (jamais plus d'une ou deux pompes en même temps)

## Configuration prévue (exemple)

```yaml
plantes:
  - nom: basilic
    adc: 0x48
    canal: 0
    relais_gpio: 17
    seuil_sec: 40        # % d'humidité sous lequel on arrose
    duree_arrosage_s: 5
    delai_min_h: 12
  - nom: menthe
    adc: 0x48
    canal: 1
    relais_gpio: 27
    seuil_sec: 45
    duree_arrosage_s: 4
    delai_min_h: 12
```

## Sécurités obligatoires

- Arrêt de toutes les pompes si le flotteur signale un réservoir vide
- Durée maximale d'arrosage par cycle, codée en dur
- Délai minimum entre deux arrosages d'une même plante
- Limite de quantité d'eau par jour et par plante
- Valeur de capteur hors plage (débranché, en court-circuit) : pas d'arrosage et alerte
- Relais au repos à l'arrêt ou au plantage du programme (pompes coupées par défaut)
- Une seule pompe active à la fois (ou deux maximum)
- Électronique dans un boîtier étanche, placée plus haut que le réservoir

## Structure du code (prévue)

```
arrosage/
├── README.md
├── config.yaml          # plantes, seuils, broches
├── requirements.txt
├── main.py              # boucle principale
├── capteurs.py          # lecture ADS1115, BME280, DS18B20, flotteur
├── pompes.py            # commande des relais et durées
├── logique.py           # décision d'arrosage et sécurités
├── stockage.py          # journalisation SQLite
├── notifications.py     # alertes
└── arrosage.service     # unité systemd
```

## Dépendances logicielles prévues

- Python 3
- `adafruit-circuitpython-ads1x15`
- `adafruit-circuitpython-bme280`
- `gpiozero` (ou `RPi.GPIO`)
- `PyYAML`
- `Flask` (optionnel, interface web)

## Budget indicatif

- Prototype 1 plante : 40 à 60 €
- 10 plantes : 150 à 250 € selon les pompes

## Suivi

| Étape | Statut |
|---|---|
| Commande du matériel | à faire |
| Prototype 1 plante | à faire |
| Journalisation et supervision | à faire |
| Passage à 10 plantes | à faire |
