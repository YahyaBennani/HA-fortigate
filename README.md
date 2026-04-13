# Haute Disponibilité (HA) sur FortiGate


## 🔑 Concepts Clés

| Terme | Définition |
|---|---|
| **HA (High Availability)** | Mécanisme de redondance pour assurer la continuité de service |
| **Failover** | Basculement automatique vers l'équipement de secours en cas de panne |
| **Heartbeat** | Lien de communication entre les membres du cluster |
| **Session Pickup** | Synchronisation des sessions TCP actives entre firewalls |
| **Override** | Paramètre permettant au Primary de reprendre son rôle après redémarrage |
| **Split-brain** | Scénario où les deux firewalls se croient Primary simultanément |
| **GARP** | Paquets utilisés pour mettre à jour les tables ARP lors d'un failover |

---

## 🏗️ Architecture du TP

```
LAN (192.168.1.0/24)
    PC1: 192.168.1.10/24
    PC2: 192.168.1.20/24
         |
    [SW2 - e0/0 à e0/3]
         |
    ┌────┴────┐
    │         │
[PrimaryFW]  [SecondaryFW]
192.168.5.133/24   192.168.5.133/24
port2 (LAN): 192.168.1.100   
port1 (WAN): DHCP
port3/port4: Heartbeat
    │
[SW1 - WAN]
192.168.5.0/24
```

**Composants :**
- PrimaryFW : Firewall maître (Priority 100)
- SecondaryFW : Firewall de secours (Priority 50)
- Liens Heartbeat : port3 + port4 (redondance)

---

## ⚙️ Configuration — Mode Active-Passive

### 1. Hostname
```
config system global
  set hostname PrimaryFW
end
```

### 2. Interface LAN (port2)
```
config system interface
  edit port2
    set ip 192.168.1.100/24
    set allowaccess ping http https
  next
end
```

### 3. DNS
```
config system dns
  set primary 8.8.8.8
end
```

### 4. Route par défaut
```
config router static
  edit 1
    set gateway 192.168.5.1
    set device port1
  next
end
```

### 5. Policy LAN → WAN avec NAT
```
config firewall policy
  edit 1
    set name LAN_to_WAN
    set srcintf port2
    set dstintf port1
    set srcaddr all
    set dstaddr all
    set action accept
    set schedule always
    set service ALL
    set nat enable
  next
end
```

### 6. Configuration HA — Primary
| Paramètre | Valeur |
|---|---|
| Mode | Active-Passive |
| Device Priority | 100 |
| Group name | HAG |
| Password | ***** |
| Session Pickup | Activé |
| Monitor interfaces | port1 (WAN) |
| Heartbeat interfaces | port3, port4 |

### 7. Configuration HA — Secondary
| Paramètre | Valeur |
|---|---|
| Mode | Active-Passive |
| Device Priority | **50** |
| Group name | HAG (identique) |
| Password | identique |
| Override | Désactivé |

> ⚠️ **Important :** Les interfaces heartbeat ne doivent PAS avoir d'adresse IP configurée manuellement. FortiGate gère cela automatiquement.

---

## 🔄 Processus de Synchronisation

Lors de la jonction du SecondaryFW au cluster :

1. Détection du cluster HA via paquets heartbeat (port3/port4)
2. Vérification de l'identité (group-name + password)
3. Synchronisation des fichiers externes (certificats SSH/VPN)
4. Synchronisation de la configuration complète (interfaces, routes, policies)
5. Attribution des rôles (Primary / Secondary)

---

## 🧪 Test de Failover — Active-Passive

### Étape 1 : Ping continu depuis PC client
```
ping 8.8.8.8 -t
```

### Étape 2 : Simuler la panne du Primary
```
execute shutdown
```

### Résultats observés
- Interruption du ping : **~12 paquets perdus (≈ 2-4 secondes)**
- SecondaryFW prend automatiquement le rôle **Primary**
- Mise à jour des tables ARP via **paquets GARP**
- Le cluster fonctionne en mode "dégradé" avec un seul membre

### Vérification post-failover
```
get system ha status
```
→ `System > HA` : SecondaryFW affiche "Role: Primary"

---

## 🔁 Réintégration du Primary

### Séquence de réintégration :
| Phase | Description |
|---|---|
| 1. Démarrage | PrimaryFW initialise ses interfaces ; SecondaryFW continue comme Primary |
| 2. Détection | PrimaryFW détecte les heartbeats et identifie le cluster |
| 3. Synchronisation | PrimaryFW compare et synchronise sa config avec le cluster |

### Override
```
config system ha
  set override enable
end
```

| | Override Activé | Override Désactivé |
|---|---|---|
| Comportement | Primary reprend son rôle automatiquement | Primary reste Secondary |
| Avantage | Cohérence, prévisibilité | Pas de 2ème coupure |
| Inconvénient | 2ème interruption (2-3 sec) | Rôles non conformes aux noms |
| Risque | Flapping si problème intermittent | — |

---
## config ip webterm (unix like)
```
sudo ip addr add 192.168.1.10/24 dev eth0
sudo ip route add default via 192.168.1.1
```
## ⚙️ Configuration — Mode Active-Active

Seul le paramètre `mode` change :

```
config system ha
  set mode a-a    ← (au lieu de a-p)
end
```

Les autres paramètres restent identiques (priority, session-pickup, heartbeat).

### Différences Active-Active vs Active-Passive

| Critère | Active-Passive | Active-Active |
|---|---|---|
| Utilisation des ressources | 50% (un seul actif) | 100% (deux actifs) |
| Capacité de traitement | 1x | 2x |
| Complexité | Faible | Élevée |
| Sessions TCP | ✅ Bien supportées | ✅ Bien supportées |
| Sessions UDP | ✅ Bien supportées | ⚠️ Support limité |
| Cas d'usage | PME, simplicité | Grande entreprise, performance |
| Troubleshooting | Simple | Complexe |

---

## 🔍 Rôle des Paramètres HA

### Heartbeat (port3 + port4)
- Surveillance mutuelle de l'état de santé
- Échange des informations de configuration et sessions
- **Deux interfaces = redondance** → évite le split-brain

### Session Pickup
- Synchronise les sessions TCP actives
- Lors d'un failover : connexions SSH/HTTP continuent sans réinitialisation
- ⚠️ Consomme de la bande passante sur le lien heartbeat

### Monitor Interface (port1 - WAN)
- Si le Primary perd la connectivité WAN → failover automatique déclenché

