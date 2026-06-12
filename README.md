# Sécurisation d'une Infrastructure Réseau Multisite — DMVPN & Firewalls Open-Source

Projet de Fin d'Études (PFE) — Infrastructure Digitale, option Réseaux et Systèmes
**OFPPT — 2024/2026**

Auteur : **Mohamed Ech-Chafiy**

---

## 📌 Contexte et problématique

À l'ère de la transformation digitale, les entreprises décentralisent leurs activités et doivent interconnecter leurs sites distants à travers Internet, un réseau public non sécurisé. Ce projet répond à la question suivante :

> Comment interconnecter efficacement et à moindre coût les sites distants de **Casablanca** et de **Rabat** avec un serveur central, tout en garantissant une isolation totale, une confidentialité absolue des flux et une protection contre les interceptions de données et les intrusions externes ?

---

## 🎯 Objectifs du projet

- **Routage dynamique OSPF** entre le cœur de réseau (R-HUB) et les routeurs d'accès (R-CASA / R-RABAT).
- **DMVPN (Dynamic Multipoint VPN)** : combinaison de **mGRE**, **NHRP** et **IPsec/ESP** pour sécuriser les communications multisites de bout en bout.
- **Architecture Multi-Firewalls en cascade** (Défense en profondeur) avec **pfSense** (firewall périphérique + NAT) et **OPNsense** (protection de la DMZ).
- **Validation et audit** du trafic chiffré via **Wireshark**.

---

## 🏗️ Architecture globale

L'infrastructure est organisée en plusieurs niveaux de confiance :

1. **Niveau Applicatif / DMZ** : serveur de production `SRV(WEB)` isolé derrière `Switch4` et `opnsense-1`.
2. **Niveau Sécurité Périmétrique (Cascade)** : `opnsense-1` (protection DMZ) → Cloud ISP → `pf-1` (firewall périphérique).
3. **Cœur de Routage (Core Layer)** : `R-HUB`, point pivot reliant la sécurité amont aux liaisons WAN.
4. **Niveau WAN / Distribution** : liaisons point-à-point publiques `212.12.12.0/30` et `212.12.12.4/30`.
5. **Niveau Accès Local (Sites distants)** :
   - **CASA** : `R-CASA` + `Switch2` + client `Win7-1`
   - **RABAT** : `R-RABAT` + `Switch1` + client `Kali-1`

> 📷 *Ajoutez ici une capture de la topologie GNS3 (`topology/gns3-topology.png`)*

---

## 🌐 Plan d'adressage IP

| Zone Réseau | Sous-Réseau IP | Passerelle / Interfaces clés |
|---|---|---|
| LAN CASA | `192.168.1.0/24` | 192.168.1.1 (R-CASA f1/0) |
| LAN RABAT | `192.168.2.0/24` | 192.168.2.1 (R-RABAT f1/0) |
| WAN R-CASA | `212.12.12.0/30` | 212.12.12.1 (R-HUB f2/0) / .2 (R-CASA) |
| WAN R-RABAT | `212.12.12.4/30` | 212.12.12.5 (R-HUB f1/1) / .6 (R-RABAT) |
| Interco OPNsense-ISP | `10.10.10.0/30` | .1 (ISP) / .2 (opnsense-1 e1) |
| Interco ISP-pfSense | `10.10.10.4/30` | .5 (ISP) / .6 (pf-1 e1) |
| Interco pfSense-R-HUB | `10.10.10.8/30` | .10 (pf-1 e2) / .9 (R-HUB f0/0) |
| DMZ Serveur Web | `192.168.100.0/24` | 192.168.100.1 (opnsense-1 e2) |

- **/30** pour les liaisons WAN (point-à-point, optimisation des adresses)
- **/24** pour les LAN et la DMZ (scalabilité)

---

## 🔐 Technologies utilisées

| Catégorie | Technologie |
|---|---|
| Routage dynamique | OSPF (Area 0 / Backbone) |
| VPN multisite | DMVPN (mGRE + NHRP) |
| Chiffrement | IPsec — ESP (AES, SHA, PSK, IKEv1) |
| Firewall périphérique | pfSense (NAT, filtrage initial) |
| Firewall DMZ | OPNsense (DNS Overrides, filtrage applicatif) |
| Serveur Web | Apache2 + VirtualHost HTTPS (Ubuntu Server) |
| Émulation | GNS3 + VMware Workstation |
| Audit réseau | Wireshark |

---

## ⚙️ Configuration — Aperçu

### OSPF (Area 0)
```
router ospf 1
 router-id 1.1.1.1
 network 1.1.1.1 0.0.0.0 area 0
 network 10.0.0.0 0.0.0.255 area 0
 network 10.10.10.8 0.0.0.3 area 0
 network 212.12.12.0 0.0.0.3 area 0
 network 212.12.12.4 0.0.0.3 area 0
```

### IPsec (Phase 1 & 2)
```
crypto isakmp policy 10
 encr aes
 authentication pre-share
 group 5
crypto isakmp key <PSK> address 0.0.0.0

crypto ipsec transform-set TS_DMVPN esp-aes
 mode transport

crypto ipsec profile IPSEC_PROFILE
 set transform-set TS_DMVPN
```

### DMVPN (Tunnel0 — R-HUB)
```
interface Tunnel0
 ip address 10.0.0.1 255.255.255.0
 no ip redirects
 ip mtu 1400
 ip nhrp map multicast dynamic
 ip nhrp network-id 1
 ip tcp adjust-mss 1360
 ip ospf network point-to-multipoint
 tunnel source Loopback0
 tunnel mode gre multipoint
 tunnel protection ipsec profile IPSEC_PROFILE
```

> 📁 Les configurations complètes des trois routeurs (R-HUB, R-CASA, R-RABAT) sont disponibles dans `configs/`.

---

## 🧱 Matrice des flux et règles de filtrage

Politique de filtrage stricte (Implicit Deny par défaut) sur l'interface périphérique :

| Règle | Source | Destination | Port | Action |
|---|---|---|---|---|
| 1 | NET_RABAT | SRV_WEB | 8443 | ✅ Autorisé |
| 2 | NET_RABAT | SRV_WEB | 8543 | ❌ Bloqué |
| 3 | NET_CASA | SRV_WEB | 8543 | ✅ Autorisé |
| 4 | NET_CASA | SRV_WEB | 8443 | ❌ Bloqué |
| 5 | * | * | ICMP | ✅ Autorisé (diagnostic) |

➡️ Chaque ville n'accède au serveur web que via **son port HTTPS dédié** :
- Rabat → `https://rabat.ma:8443`
- Casablanca → `https://casablanca.ma:8543`

---

## 🖥️ Durcissement du serveur Web (SRV-WEB)

- **Adressage IP statique** via Netplan (`/etc/netplan/99-custom-config.yaml`)
- **VirtualHosts Apache2 HTTPS** (certificats auto-signés) :
  - `rabat.ma.conf` → port `8443`
  - `casablanca.ma.conf` → port `8543`
- **Résolution DNS locale** via **Unbound DNS Overrides** sur OPNsense (`rabat.ma`, `casablanca.ma` → `192.168.100.250`)

---

## ✅ Validation et tests

- **OSPF** : `show ip ospf neighbor` → adjacences `FULL` confirmées entre R-HUB, R-CASA et R-RABAT.
- **IPsec Phase 1** : `show crypto isakmp sa` → état `QM_IDLE` actif sur tous les sites.
- **IPsec Phase 2** : `show crypto ipsec sa` → compteurs de chiffrement/déchiffrement sans erreur (`#send errors 0`, `#recv errors 0`).
- **Services Web** : accès HTTPS validé depuis Rabat (port 8443) et Casablanca (port 8543), avec blocage croisé confirmé.
- **Wireshark** :
  - Avant chiffrement : trafic OSPF, SSH et ICMP visibles en clair.
  - Après activation du DMVPN/IPsec : seul le protocole **ESP (IP 50)** est visible sur le WAN, preuve du chiffrement total des flux.

---

## 📈 Perspectives d'évolution

- Mise en place d'un cluster de pare-feux en **Haute Disponibilité (HA)**
- Déploiement d'une supervision centralisée avec **Zabbix**

---

## 📂 Structure du repository

```
├── README.md
├── docs/
│   └── Rapport-PFE.pdf
├── configs/
│   ├── R-HUB.txt
│   ├── R-CASA.txt
│   └── R-RABAT.txt
├── apache-vhosts/
│   ├── rabat.ma.conf
│   └── casablanca.ma.conf
├── netplan/
│   └── 99-custom-config.yaml
└── topology/
    └── gns3-topology.png
```

---

## 📜 Licence

Projet académique réalisé dans le cadre du PFE OFPPT — © 2026 Mohamed Ech-Chafiy
