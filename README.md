# GroundSat — Nanosatellite Ground Emulation Platform

GroundSat est l'équivalent électronique et logiciel d'un vrai nanosatellite, testé au sol. Un microcontrôleur embarqué (Raspberry Pi Pico W ou ESP32) joue le rôle du satellite : il lit des capteurs physiques, encode les données dans un protocole de télémétrie binaire, et les transmet par radio LoRa. De l'autre côté, une station sol reçoit, décode et affiche ces données en temps réel — exactement comme une vraie station de contrôle au sol.

Le projet ne simule pas le comportement orbital, il réplique la chaîne complète d'un CubeSat réel : acquisition → encodage → transmission → réception → décodage → visualisation.

---