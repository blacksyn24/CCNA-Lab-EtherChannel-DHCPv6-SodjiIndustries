# 🔗 SedaLab-03 — EtherChannel LACP & Double-Stack IPv4/IPv6 (Sodji Industries)

![Cisco](https://img.shields.io/badge/Cisco-CCNA-blue?style=for-the-badge&logo=cisco&logoColor=white)
![PacketTracer](https://img.shields.io/badge/Packet%20Tracer-8.x-orange?style=for-the-badge&logo=cisco)
![Status](https://img.shields.io/badge/Status-✅%20Completed-brightgreen?style=for-the-badge)
![Lab](https://img.shields.io/badge/Série-Entreprise%20Moyenne-purple?style=for-the-badge)
![IPv6](https://img.shields.io/badge/IPv6-DHCPv6%20%2B%20SLAAC-red?style=for-the-badge)

---

## 📋 Description

Troisième lab de la série progressive d'infrastructures d'entreprise. Ce lab simule **Sodji Industries**, une entreprise moyenne nécessitant des liens à haute disponibilité entre le cœur de réseau et un switch d'accès (EtherChannel LACP), ainsi qu'un déploiement double-stack IPv4/IPv6 pour le service IT, combinant SLAAC et DHCPv6 stateless.

### Objectifs
- ✅ Configurer un **EtherChannel LACP** (Port-channel) entre deux switches
- ✅ Activer **ipv6 unicast-routing** sur le routeur
- ✅ Déployer le **double-stack IPv4/IPv6** sur une sous-interface
- ✅ Combiner **SLAAC** et **DHCPv6 stateless** pour la distribution DNS
- ✅ Vérifier la répartition de charge sur l'agrégat de liens

---

## 📖 Théorie

### Pourquoi l'EtherChannel ?

Sans EtherChannel                  Avec EtherChannel (LACP)
──────────────────                 ─────────────────────────
1 seul lien actif                  Plusieurs liens actifs simultanément
STP bloque les liens redondants    STP voit un seul lien logique (Po1)
Pas de tolérance de panne          Bande passante agrégée + tolérance

### SLAAC vs DHCPv6 stateless vs stateful

SLAAC (Stateless Address Autoconfig)
→ Le PC génère lui-même son adresse IPv6 à partir du préfixe annoncé (RA)
DHCPv6 stateless (M=0, O=1)
→ Adresse via SLAAC + infos complémentaires (DNS) via DHCPv6
→ Configuré ici avec "ipv6 nd other-config-flag"
DHCPv6 stateful (M=1)
→ Le serveur DHCPv6 attribue TOUT (adresse + DNS), comme en IPv4

---

## 🖥️ Équipements

| Équipement | Modèle | Nom | Rôle |
|---|---|---|---|
| 🔁 Switch L3 | Cisco 3560-24PS | SW-CORE | Cœur de réseau |
| 🔁 Switch L2 | Cisco 2960-24TT | SW-ACCESS | Accès (lien EtherChannel) |
| 🔀 Routeur | Cisco 2911 | R1 | Double-stack + DHCPv6 |

---

## 🗺️ Topologie

<img width="1920" height="1080" alt="ImageTopologie" src="https://github.com/user-attachments/assets/8264ead7-5ffc-4f47-9f0e-d994b46a8436" />

VLAN 30 (IT)   VLAN 40 (Production)
8 postes       6 postes
double-stack   IPv4 seul



---

## 📊 Plan d'adressage

| VLAN | Nom | Réseau IPv4 | Réseau IPv6 | Passerelle |
|---|---|---|---|---|
| 30 | IT | 10.30.30.0/24 | 2001:db8:30::/64 | 10.30.30.1 / ::1 |
| 40 | PRODUCTION | 10.30.40.0/24 | — | 10.30.40.1 |
| 99 | MANAGEMENT | 10.30.99.0/24 | — | 10.30.99.1 |

---

## 🔌 Câblage

| Source | Ports | Destination | Ports | Type |
|---|---|---|---|---|
| SW-CORE | Gi0/1, Gi0/2 | SW-ACCESS | Gi0/1, Gi0/2 | EtherChannel LACP (Po1) |
| SW-CORE | **Fa0/3** | R1 | Gi0/0 | Trunk router-on-a-stick |
| SW-ACCESS | Fa0/2-9 | PC-IT1 à 8 | Fa0 | Accès VLAN 30 |
| SW-ACCESS | Fa0/10-15 | PC-Prod1 à 6 | Fa0 | Accès VLAN 40 |

> ⚠️ Le switch 3560-24PS ne dispose que de 2 ports Gigabit (Gi0/1, Gi0/2), déjà utilisés par l'EtherChannel. Le lien vers R1 passe donc par un port **FastEthernet** (Fa0/3).

---

## ⚙️ Configuration complète

### 🔧 Task 1 — SW-CORE : VLAN + EtherChannel + lien vers R1

```cisco
enable
configure terminal
hostname SW-CORE

vlan 30
 name IT
vlan 40
 name PRODUCTION
vlan 99
 name MANAGEMENT
exit

interface range gigabitEthernet 0/1 - 2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 99
 channel-group 1 mode active
exit

interface port-channel 1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 99
exit

interface fastEthernet 0/3
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 99
exit
```

### 🔧 Task 2 — SW-ACCESS : VLAN + EtherChannel + ports d'accès

```cisco
enable
configure terminal
hostname SW-ACCESS

vlan 30
 name IT
vlan 40
 name PRODUCTION
vlan 99
 name MANAGEMENT
exit

interface range gigabitEthernet 0/1 - 2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 99
 channel-group 1 mode active
exit

interface port-channel 1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 99
exit

interface range fastEthernet 0/2 - 9
 switchport mode access
 switchport access vlan 30
exit

interface range fastEthernet 0/10 - 15
 switchport mode access
 switchport access vlan 40
exit
```

### 🔧 Task 3 — R1 : double-stack + DHCPv4/v6

```cisco
enable
configure terminal
hostname R1
ipv6 unicast-routing

interface gigabitEthernet 0/0
 no shutdown
exit

interface gigabitEthernet 0/0.30
 encapsulation dot1Q 30
 ip address 10.30.30.1 255.255.255.0
 ipv6 address 2001:db8:30::1/64
 ipv6 nd other-config-flag
exit

interface gigabitEthernet 0/0.40
 encapsulation dot1Q 40
 ip address 10.30.40.1 255.255.255.0
exit

interface gigabitEthernet 0/0.99
 encapsulation dot1Q 99 native
 ip address 10.30.99.1 255.255.255.0
exit

ip dhcp pool IT
 network 10.30.30.0 255.255.255.0
 default-router 10.30.30.1
exit

ip dhcp pool PRODUCTION
 network 10.30.40.0 255.255.255.0
 default-router 10.30.40.1
exit

ipv6 dhcp pool IT-V6
 dns-server 2001:4860:4860::8888
 domain-name sodji.local
exit

interface gigabitEthernet 0/0.30
 ipv6 dhcp server IT-V6
exit
end
write
```

---

## 🧪 Tests finaux

```cisco
SW-CORE# show etherchannel summary
SW-CORE# show interfaces port-channel 1
R1# show ipv6 interface brief
PC-IT1> ping 10.30.30.1        ✅
PC-IT1> ping ipv6 2001:db8:30::1  ✅
PC-Prod1> ping 10.30.40.1      ✅
```

---

## 💡 Points clés

| 🔑 Commande | 📖 Rôle |
|---|---|
| `channel-group 1 mode active` | Active LACP sur l'interface membre |
| `ipv6 unicast-routing` | Active le routage IPv6 + envoi des RA |
| `ipv6 nd other-config-flag` | Combine SLAAC + DHCPv6 stateless |
| `show etherchannel summary` | Voir l'état de l'agrégat |
| `show ipv6 interface brief` | Voir les adresses IPv6 configurées |

---

## 📊 Comparatif Labs

| | Lab 2 (SVI + STP) | Lab 3 (EtherChannel + IPv6) |
|---|---|---|
| **Redondance** | STP (liens bloqués) | LACP (liens actifs agrégés) |
| **IP** | IPv4 seul | Double-stack IPv4/IPv6 |
| **Bande passante** | 1 Gbps par lien | 2 Gbps agrégés |
| **Nouveauté** | Routage L3 natif | SLAAC + DHCPv6 |

---

## 👨‍💻 Auteur

**LANDJIDE Urbain Sedami**
🎓 Étudiant en 2ème année — Licence Professionnelle
📡 Réseaux Informatique Mobilité Sécurité (RMS)
🏫 Cisco Networking Academy — Préparation CCNA
📍 Cotonou, Bénin 🇧🇯

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connecter-blue?style=flat-square&logo=linkedin)](https://www.linkedin.com/in/urbain-sedami-landjide-9b49043a8/)

---

## 📄 Licence

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)











