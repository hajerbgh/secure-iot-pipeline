# Secure IoT Pipeline : MQTT sur mTLS, Telegraf, InfluxDB, Grafana

Pipeline de télémétrie IoT de bout en bout, avec **démonstration d'attaque sur MQTT en clair** puis **durcissement par PKI et TLS mutuel (mTLS)**.

`capteur (simulé) → Mosquitto (MQTT/mTLS) → Telegraf → InfluxDB → Grafana`

![Architecture](docs/architecture.png)

## Objectif

Montrer concrètement pourquoi MQTT sans chiffrement ni authentification est dangereux (écoute passive, injection de fausses mesures), puis le corriger et valider le résultat par une matrice avant/après.

## Fonctionnalités

- Broker Eclipse Mosquitto : port **1883** (baseline vulnérable) et port **8883** (mTLS, `require_certificate true`)
- PKI locale générée avec OpenSSL (CA, certificat serveur avec SAN, certificat client)
- Collecte par Telegraf (`mqtt_consumer` en TLS), stockage dans InfluxDB v2, dashboard Grafana
- Modèle de menaces **STRIDE** et matrice de validation avant/après
- Secrets hors du dépôt (`.env`, certificats générés localement)

## Démarrage rapide

**Prérequis :** Docker, Docker Compose v2, OpenSSL.

```bash
git clone https://github.com/<ton-compte>/secure-iot-pipeline.git
cd secure-iot-pipeline

# 1. Variables d'environnement
cp .env.example .env            # puis édite les valeurs

# 2. Génère la PKI (CA, serveur, client)
./scripts/gen-certs.sh

# 3. Démarre InfluxDB et Grafana
docker compose up -d influxdb grafana
```

4. Ouvre InfluxDB (`http://localhost:8086`, utilisateur `admin`, mot de passe de `.env`) puis **Load Data → API Tokens** et crée un jeton **Write** sur le bucket `iot` (Telegraf) et un jeton **Read** (Grafana).
5. Mets le jeton Write dans `.env` (`INFLUX_TOKEN=...`), puis :

```bash
docker compose up -d
```

6. Dans Grafana (`http://localhost:3000`) : **Connections → Data sources → InfluxDB**, URL `http://influxdb:8086`, organisation `lab`, bucket `iot`, jeton Read.

## Tester le pipeline

Publie une mesure avec le certificat client :

```bash
docker exec mosquitto-baseline mosquitto_pub -h localhost -p 8883 \
  --cafile /mosquitto/config/certs/ca.crt \
  --cert /mosquitto/config/certs/client.crt \
  --key /mosquitto/config/certs/client.key \
  -t iot/esp32-01/telemetry -m '{"temp": 25.5}'
```

La mesure apparaît dans InfluxDB (Data Explorer, bucket `iot`) puis dans Grafana.

## Scénario d'attaque et résultats

| Scénario | Baseline (1883) | Version durcie (8883, mTLS) |
|---|---|---|
| Écoute passive (Wireshark) | Payload JSON lisible en clair | Trafic chiffré, inexploitable |
| Connexion au broker | Anonyme et ouverte | Refusée sans certificat signé par la CA du projet |
| Injection de `{"temp": 999.9}` | Acceptée et visible dans Grafana | Bloquée au handshake TLS |
| Flux légitime (certificat valide) | Remontée brute | Stocké dans InfluxDB et affiché dans Grafana |

Captures dans [`docs/screenshots/`](docs/screenshots/) :

## Modèle de menaces (STRIDE)

| Menace | Mesure | Statut |
|---|---|---|
| Spoofing | Certificats X.509 signés par la CA (mTLS) | Partiel : certificat client partagé |
| Tampering | TLS (intégrité en transit) | Couvert en transit uniquement |
| Information Disclosure | Trafic chiffré sur 8883 | Couvert |
| Denial of Service | `allow_anonymous false`, handshake obligatoire | Partiel |
| Elevation of Privilege | Aucune (pas d'ACL) | Non couvert |
| Repudiation | Aucune (certificat partagé) | Non couvert |

## Limites et améliorations

Ce projet valide le **chiffrement et l'authentification du transport**. Il ne prétend pas couvrir plus :

- **Pas d'ACL** : un client authentifié peut lire et écrire sur n'importe quel topic. Piste : `use_identity_as_username true` et un fichier ACL (`pattern write iot/%u/telemetry`).
- **Certificat client partagé** : on ne distingue pas les appareils et on ne peut pas en révoquer un seul. Piste : un certificat par appareil, avec le CN comme identité.
- **Clients simulés** (`mosquitto_pub`) : l'ESP32 physique n'est pas intégré.
- **Intégrité applicative** : un client légitime compromis peut envoyer une mauvaise valeur. Piste : validation de plage côté Telegraf.
- **Stockage des clés** : secure element ou flash encryption sur du matériel réel.
- **Cycle de vie des certificats** : pas de rotation ni de révocation automatisées.
- **Réseau interne** : Telegraf, InfluxDB et Grafana communiquent en HTTP dans un réseau Docker privé.

## Sécurité du dépôt

Les clés, certificats et le fichier `.env` sont exclus par `.gitignore`. Les certificats se régénèrent avec `scripts/gen-certs.sh`. Le mode de fonctionnement du labo n'est pas prévu pour la production.

## Technologies

Eclipse Mosquitto 2.0, Telegraf 1.30, InfluxDB 2.7, Grafana, Docker Compose, OpenSSL, Kali Linux et Wireshark (démonstration d'attaque).

## Auteure

**Hajer Benghazi**, étudiante en 4ᵉ année d'ingénierie informatique, Faculté des Sciences de Tunis. LinkedIn : `<ton-lien>`

## Licence

MIT

## Captures

| Fichier | Contenu |
|---|---|
| `injectiondekaliaubuntu.png`, `injectiondekaliaubuntu2.png` | Injection depuis Kali (baseline) |
| `tentativeKali.png` | Tentative d'accès après durcissement |
| `generationdes_cles.png` | Génération de la PKI |
| `mtls.png` | Configuration mTLS |
| `dashbo.png`, `result.png` | Grafana |
| `DATAcreation.png` | Source de données InfluxDB |

Toutes sont dans [`docs/screenshots/`](docs/screenshots/).
