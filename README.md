# GroundSat — Nanosatellite Ground Emulation Platform

GroundSat est l'équivalent électronique et logiciel d'un vrai nanosatellite, testé au sol. Un microcontrôleur embarqué (Raspberry Pi Pico W ou ESP32) joue le rôle du satellite : il lit des capteurs physiques, encode les données dans un protocole de télémétrie binaire, et les transmet par radio LoRa. De l'autre côté, une station sol reçoit, décode et affiche ces données en temps réel — exactement comme une vraie station de contrôle au sol.

Le projet ne simule pas le comportement orbital, il réplique la chaîne complète d'un CubeSat réel : acquisition → encodage → transmission → réception → décodage → visualisation.

---

## Hypothèses de conception

- **Le lien radio est le point critique.** La communication LoRa entre deux modules est le coeur du projet. Tout le reste (capteurs, dashboard) en dépend. On valide d'abord la liaison avant d'y greffer les données.
- **Le protocole de télémétrie est binaire.** Les données sont encodées en trames compactes avec `struct.pack` et un CRC16, comme sur un vrai satellite. Pas de JSON, pas de CSV — chaque bit compte dans une liaison radio contrainte.
- **L'autonomie énergétique est une contrainte réelle.** Le module satellite tourne sur batterie LiPo avec panneau solaire optionnel. Le firmware gère le deep sleep entre les transmissions pour maximiser l'autonomie.
- **La station sol est passive.** Elle reçoit, décode et enregistre. Elle n'envoie pas de commande vers le satellite en v1 (pas de télécommande montante).
- **Deux environnements de code distincts.** Le firmware embarqué est en MicroPython (Pico W) ou Arduino C++ (ESP32). Le code station sol est en Python 3. Les deux partagent uniquement la définition du format de trame.

---

## Matériel

### Côté satellite (embarqué)

| Composant | Rôle |
|---|---|
| Raspberry Pi Pico W / ESP32 | Microcontrôleur principal — lecture capteurs, encodage, transmission |
| BME280 / BME680 | Température, pression, humidité (+ qualité d'air VOC pour BME680) |
| LTR-390 | Luminosité ambiante + rayonnement UV |
| VEML6075 | Index UV précis (UVA + UVB) |
| Anémomètre + girouette | Vitesse et direction du vent (sortie analogique ou impulsions) |
| MPU-6050 / ICM-42688 | IMU — accéléromètre + gyroscope (simulation ADCS CubeSat) |
| Module LoRa SX1276 (Ra-02) | Transmission radio 433 MHz / 868 MHz |
| Batterie LiPo 3.7V + TP4056 | Alimentation et gestion de charge |
| Panneau solaire 5V (optionnel) | Recharge autonome — simulation alimentation orbitale |

### Côté station sol

| Composant | Rôle |
|---|---|
| ESP32 ou Raspberry Pi | Récepteur LoRa branché au PC via USB |
| Module LoRa SX1276 (Ra-02) | Réception radio — même fréquence que le satellite |
| PC (Linux / macOS / Windows) | Décodage, stockage, visualisation des trames reçues |

---

## Architecture logicielle

```
GroundSat/
├── satellite/
│   ├── main.py                 → Boucle principale : réveil, lecture, encodage, transmission, sleep
│   ├── sensor_reader.py        → Lecture de tous les capteurs via I2C / SPI / ADC
│   ├── imu_reader.py           → Lecture gyroscope + accéléromètre (MPU-6050)
│   ├── power_manager.py        → Deep sleep, mesure tension batterie, budget énergétique
│   └── lora_transmitter.py     → Envoi de trame LoRa, gestion RSSI, retry
├── telemetry/
│   ├── frame_format.py         → Définition du format de trame (partagé satellite / sol)
│   ├── frame_encoder.py        → struct.pack → trame binaire + CRC16
│   └── frame_decoder.py        → Réception trame → vérification CRC → dict de valeurs
├── ground_station/
│   ├── gs_receiver.py          → Écoute port série, réception trames brutes, RSSI / SNR
│   ├── db_writer.py            → Écriture en base de données (InfluxDB ou SQLite)
│   └── link_monitor.py         → Calcul packet loss, latence, qualité de liaison
├── dashboard/
│   ├── app.py                  → Dashboard Dash/Plotly — affichage temps réel
│   ├── plots.py                → Graphes capteurs, carte de signal, orientation IMU
│   └── alerts.py               → Seuils d'alerte, export CSV des sessions
├── docs/
│   ├── hardware_setup.md       → Schémas de câblage, pinouts, fréquences LoRa
│   ├── frame_protocol.md       → Spécification complète du protocole de télémétrie
│   └── energy_budget.md        → Calcul d'autonomie, mesures de consommation
└── tests/
    ├── test_telemetry.py        → encode → decode idempotent, CRC fail, valeurs limites
    ├── test_sensor_reader.py   → Mocks I2C, valeurs hors plage, capteur absent
    └── test_link_monitor.py    → Calcul packet loss, détection perte de signal
```

---

## Protocole de télémétrie

Le format de trame est la pièce centrale du projet. Il est défini une seule fois dans `telemetry/frame_format.py` et partagé entre le firmware satellite et le code station sol.

### Structure d'une trame (22 bytes)

```
Offset  Taille  Champ           Type        Description
------  ------  -----           ----        -----------
0       2B      header          uint16      0xAA55 — marqueur de début de trame
2       4B      timestamp       uint32      Secondes depuis boot du satellite
6       2B      temperature     int16       Température × 100 (°C, ex: 2350 = 23.50°C)
8       2B      pressure        uint16      Pression - 900 hPa × 10 (ex: 130 = 913.0 hPa)
10      1B      humidity        uint8       Humidité relative % (0–100)
11      1B      uv_index        uint8       Index UV × 10 (ex: 35 = 3.5)
12      2B      wind_speed      uint16      Vitesse vent × 100 (m/s, ex: 520 = 5.20 m/s)
14      1B      wind_direction  uint8       Direction vent en degrés / 2 (ex: 90 = 180°)
15      2B      gyro_x          int16       Gyroscope X × 100 (°/s)
17      2B      gyro_y          int16       Gyroscope Y × 100 (°/s)
19      2B      gyro_z          int16       Gyroscope Z × 100 (°/s)
21      2B      crc16           uint16      CRC16-CCITT de l'ensemble des 20 bytes précédents
```

### Encodage / décodage

```python
# telemetry/frame_encoder.py
import struct, crcmod

FRAME_HEADER = 0xAA55
FRAME_FORMAT = ">HI hHBB HB hhh"  # big-endian, 20 bytes de données

crc16 = crcmod.predefined.mkCrcFun('crc-ccitt-false')

def encode(data: dict) -> bytes:
    payload = struct.pack(FRAME_FORMAT,
        FRAME_HEADER,
        data['timestamp'],
        int(data['temperature'] * 100),
        int((data['pressure'] - 900) * 10),
        int(data['humidity']),
        int(data['uv_index'] * 10),
        int(data['wind_speed'] * 100),
        int(data['wind_direction'] / 2),
        int(data['gyro_x'] * 100),
        int(data['gyro_y'] * 100),
        int(data['gyro_z'] * 100),
    )
    return payload + struct.pack(">H", crc16(payload))


# telemetry/frame_decoder.py
def decode(raw: bytes) -> dict | None:
    if len(raw) != 22:
        return None
    payload, received_crc = raw[:20], struct.unpack(">H", raw[20:])[0]
    if crc16(payload) != received_crc:
        return None  # Trame corrompue
    fields = struct.unpack(FRAME_FORMAT, payload)
    return {
        'timestamp':      fields[1],
        'temperature':    fields[2] / 100,
        'pressure':       fields[3] / 10 + 900,
        'humidity':       fields[4],
        'uv_index':       fields[5] / 10,
        'wind_speed':     fields[6] / 100,
        'wind_direction': fields[7] * 2,
        'gyro_x':         fields[8] / 100,
        'gyro_y':         fields[9] / 100,
        'gyro_z':         fields[10] / 100,
    }
```

---

## Boucle principale du satellite

```
1. RÉVEIL        → Sortie du deep sleep (timer périodique, ex: toutes les 30s)
2. LECTURE       → Tous les capteurs lus séquentiellement via I2C / SPI / ADC
3. ENCODAGE      → Données brutes → trame binaire 22 bytes + CRC16
4. TRANSMISSION  → Envoi LoRa, attente ACK optionnel (3 tentatives max)
5. LOG           → Écriture locale sur flash (si la transmission échoue)
6. SLEEP         → Deep sleep jusqu'au prochain cycle — LED status éteinte
```

Le budget énergétique cible est une autonomie de 72h minimum sur batterie LiPo 1000 mAh, avec un cycle de transmission toutes les 30 secondes.

---

## Gestion d'énergie

La gestion d'énergie est un enjeu de premier plan — exactement comme sur un vrai CubeSat en orbite basse.

```python
# satellite/power_manager.py
TRANSMISSION_INTERVAL_S = 30   # secondes entre chaque réveil
BATTERY_LOW_THRESHOLD_V = 3.5  # tension minimale avant réduction du cycle

def get_battery_voltage() -> float:
    raw = adc.read_u16()           # lecture ADC du diviseur de tension
    return raw / 65535 * 3.3 * 2  # pont résistif R1=R2

def enter_deep_sleep(seconds: int):
    machine.deepsleep(seconds * 1000)  # MicroPython
```

En cas de batterie faible (< 3.5V), le cycle de transmission est réduit automatiquement (toutes les 5 minutes au lieu de 30 secondes) pour prolonger l'autonomie.

---

## Communication LoRa

Les deux modules LoRa (satellite et station sol) fonctionnent sur la même fréquence et les mêmes paramètres radio.

```
Fréquence     : 868.1 MHz (Europe) ou 433.0 MHz
Spreading Factor: SF10 — bon compromis portée / débit
Bandwidth     : 125 kHz
Coding Rate   : 4/5
Sync Word     : 0x12 (réseau privé)
Tx Power      : 17 dBm (max légal Europe sans licence)
```

Le nœud récepteur station sol lit le RSSI et le SNR de chaque trame reçue, ce qui permet de calculer en continu la qualité du lien radio et le budget de liaison.

---

## Progression recommandée

### Étape 1 — Liaison LoRa brute
Valider la communication radio entre les deux modules. Un module envoie `"PING"`, l'autre répond `"PONG"`. Mesurer le RSSI à différentes distances.

**Ce qu'on code :** scripts de test LoRa minimalistes, validation du hardware

### Étape 2 — Lecture des capteurs
Lire et afficher chaque capteur individuellement sur le port série. Valider les valeurs physiques (température ambiante, pression au niveau de la mer, etc.).

**Ce qu'on code :** `sensor_reader.py`, `imu_reader.py`, tests unitaires avec mocks I2C

### Étape 3 — Protocole de télémétrie
Implémenter l'encodage binaire et le CRC16. Valider que `decode(encode(data)) == data` sur toutes les valeurs limites.

**Ce qu'on code :** `frame_encoder.py`, `frame_decoder.py`, suite de tests pytest

### Étape 4 — Première transmission complète
Le satellite lit les capteurs, encode une trame et la transmet. La station sol reçoit, vérifie le CRC et affiche les valeurs décodées dans le terminal.

**Ce qu'on code :** `lora_transmitter.py`, `gs_receiver.py`, boucle principale `main.py`

### Étape 5 — Stockage et gestion d'énergie
Écriture des trames reçues en base de données (InfluxDB). Implémentation du deep sleep et mesure d'autonomie réelle sur batterie.

**Ce qu'on code :** `db_writer.py`, `power_manager.py`, benchmark consommation

### Étape 6 — Dashboard temps réel
Interface web affichant en live les courbes de capteurs, l'orientation IMU, la qualité du lien radio et le bilan énergétique.

**Ce qu'on code :** `dashboard/app.py`, graphes Plotly, alertes sur seuils

---

## Complexités volontairement mises de côté (v1)

Ces points ont été identifiés et documentés, mais écartés pour maintenir une première version fonctionnelle :

- **Liaison montante (télécommande).** En v1, la communication est unidirectionnelle : satellite → sol. L'envoi de commandes depuis la station sol vers le satellite (changement de fréquence d'émission, reset, mode veille forcé) est prévu pour la v2.
- **Chiffrement de la trame.** Les données sont transmises en clair. L'ajout d'AES-128 sur la payload est documenté mais non implémenté (contrainte de puissance de calcul sur Pico W).
- **Correction d'erreur (FEC).** LoRa intègre un forward error correction basique. Un FEC applicatif supplémentaire (Reed-Solomon) n'est pas implémenté en v1.
- **Horodatage absolu.** Le champ `timestamp` est relatif au boot du satellite. La synchronisation avec une référence temps absolue (GPS PPS ou NTP) est prévue pour la v2.
- **Multi-satellites.** Le protocole supporte un seul émetteur. L'ajout d'un champ `satellite_id` dans le header permettrait de gérer plusieurs noeuds sur le même réseau LoRa.

---

## Reste à développer

### Liaison montante (v2)
Implémenter un canal de télécommande depuis la station sol vers le satellite — changement du cycle de transmission, activation d'un capteur spécifique, reset à distance. Nécessite un protocole request/acknowledge et une gestion des collisions radio.

### Correction d'erreur applicative
Ajouter une couche Reed-Solomon sur la payload pour récupérer des trames partiellement corrompues dans les conditions de faible signal (RSSI < -120 dBm).

### Visualisation orbite simulée
Intégrer la bibliothèque `sgp4` pour simuler la position orbitale théorique d'un CubeSat LEO (400 km) et afficher la couverture radio en fonction de l'élévation. Permet de valider le budget de liaison dans un contexte orbital réaliste.

### Cartographie du signal
Enregistrer les coordonnées GPS de la station sol et la qualité du signal (RSSI, SNR, packet loss) pour chaque session. Générer une carte de couverture radio sur un fond cartographique (Folium / Leaflet).

### Export et analyse post-mission
Exporter chaque session de télémétrie au format CSV et générer un rapport automatique (qualité liaison, statistiques capteurs, anomalies détectées). Utilisable pour valider des changements de configuration LoRa entre sessions.

---

## Installation

```bash
# Prérequis Python 3.11+
git clone https://github.com/alassane8/GroundSat.git
cd GroundSat
pip install -r requirements.txt

# Flasher le firmware sur Raspberry Pi Pico W
# Copier satellite/ sur le Pico via mpremote ou Thonny

# Lancer la station sol
python ground_station/gs_receiver.py --port /dev/ttyUSB0 --baud 115200

# Lancer le dashboard
python dashboard/app.py
# → http://localhost:8050
```

---

## Stack technique

| Couche | Technologies |
|---|---|
| Firmware satellite | MicroPython (Pico W) / Arduino C++ (ESP32) |
| Protocole télémétrie | struct binaire, CRC16-CCITT, crcmod |
| Communication radio | LoRa SX1276, 868 MHz, SF10 |
| Station sol | Python 3.11+, pyserial, asyncio |
| Stockage | InfluxDB (time-series) ou SQLite |
| Visualisation | Dash, Plotly, Pandas |
| Tests | pytest, unittest.mock (mocks I2C) |
| CI | GitHub Actions — lint + tests sur chaque push |