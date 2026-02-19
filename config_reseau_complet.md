# Configuration Réseau Complète — router-sae

## Architecture

| IP | Rôle | OS |
|---|---|---|
| 192.168.10.0/24 | VLAN 10 - ADMIN | — |
| 192.168.20.0/24 | VLAN 20 - INTERNE | — |
| 192.168.30.0/24 | VLAN 30 - DMZ | — |
| 192.168.80.0/24 | VLAN 80 - Hyperviseur | — |
| 192.168.80.10 | VM Outil (Ansible + SSH) | Debian 13 |
| 192.168.80.160 | VM Web (Nginx + SSH) | Ubuntu Server |
| 192.168.80.170 | VM BDD2 (PostgreSQL + SSH) | Alpine |
| 192.168.80.180 | VM BDD3 (PostgreSQL + SSH) | Alpine |
| 192.168.80.190 | VM BDD1 (PostgreSQL + SSH) | Alpine |
| 192.168.80.253 | Hyperviseur | — |

### Règles d'accès inter-VLANs

| Source | Destination | Service |
|---|---|---|
| VLAN 10 (ADMIN) | VM Outil (80.10) | SSH |
| VLAN 10 (ADMIN) | Hyperviseur (80.253) | SSH |
| VLAN 20 (INTERNE) | VM BDD1/2/3 | SSH |
| VLAN 30 (DMZ) | VM Web (80.160) | SSH |
| Internet | VM Web (80.160) | HTTP/HTTPS |
| VM Outil (80.10) | Toutes les VMs | SSH (Ansible) |
| VM Web (80.160) | VM BDD1/2/3 | PostgreSQL (5432) |

---

## 1. Configuration Cisco IOS — From Scratch

```cisco
conf t

! ------------------------------------------------------------
! HOSTNAME & AUTHENTIFICATION
! ------------------------------------------------------------
hostname router-sae
enable secret Password1*
username aledoskur privilege 15 secret Password1*

! ------------------------------------------------------------
! DHCP — Exclure les IPs de passerelle
! ------------------------------------------------------------
ip dhcp excluded-address 192.168.10.1
ip dhcp excluded-address 192.168.20.1
ip dhcp excluded-address 192.168.30.1
ip dhcp excluded-address 192.168.80.1
ip dhcp excluded-address 192.168.80.10
ip dhcp excluded-address 192.168.80.160
ip dhcp excluded-address 192.168.80.170
ip dhcp excluded-address 192.168.80.180
ip dhcp excluded-address 192.168.80.190
ip dhcp excluded-address 192.168.80.253

ip dhcp pool ADMIN
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 8.8.8.8

ip dhcp pool INTERNE
 network 192.168.20.0 255.255.255.0
 default-router 192.168.20.1
 dns-server 8.8.8.8

ip dhcp pool DMZ
 network 192.168.30.0 255.255.255.0
 default-router 192.168.30.1
 dns-server 8.8.8.8

ip dhcp pool HYPERVISEUR
 network 192.168.80.0 255.255.255.0
 default-router 192.168.80.1
 dns-server 8.8.8.8

! ------------------------------------------------------------
! INTERFACE WAN — Sortie Internet
! ------------------------------------------------------------
interface GigabitEthernet0/0/1
 description WAN - Internet FAI
 ip address dhcp
 ip nat outside
 no shutdown

! ------------------------------------------------------------
! INTERFACE LAN — Pas d'IP directe, les SVIs gèrent
! ------------------------------------------------------------
interface GigabitEthernet0/0/0
 description LAN - Switch principal
 no ip address
 no shutdown

! ------------------------------------------------------------
! PORTS SWITCH — ACCESS VLAN
! ------------------------------------------------------------
interface GigabitEthernet0/1/0
 description ADMIN - VLAN 10
 switchport mode access
 switchport access vlan 10
 no shutdown

interface GigabitEthernet0/1/1
 description INTERNE - VLAN 20
 switchport mode access
 switchport access vlan 20
 no shutdown

interface GigabitEthernet0/1/2
 description DMZ - VLAN 30
 switchport mode access
 switchport access vlan 30
 no shutdown

interface GigabitEthernet0/1/3
 description HYPERVISEUR - VLAN 80
 switchport mode access
 switchport access vlan 80
 no shutdown

! ------------------------------------------------------------
! INTERFACES VLAN (SVI) — Passerelles + NAT inside
! ------------------------------------------------------------
interface Vlan1
 no ip address
 shutdown

interface Vlan10
 description ADMIN
 ip address 192.168.10.1 255.255.255.0
 ip nat inside
 no shutdown

interface Vlan20
 description INTERNE
 ip address 192.168.20.1 255.255.255.0
 ip nat inside
 no shutdown

interface Vlan30
 description DMZ
 ip address 192.168.30.1 255.255.255.0
 ip nat inside
 no shutdown

interface Vlan80
 description HYPERVISEUR
 ip address 192.168.80.1 255.255.255.0
 ip nat inside
 no shutdown

! ------------------------------------------------------------
! NAT — Tous les VLANs sortent sur l'IP WAN
! ------------------------------------------------------------
ip access-list standard NAT_ALL_VLANS
 10 permit 192.168.10.0 0.0.0.255
 20 permit 192.168.20.0 0.0.0.255
 30 permit 192.168.30.0 0.0.0.255
 40 permit 192.168.80.0 0.0.0.255

ip nat inside source list NAT_ALL_VLANS interface GigabitEthernet0/0/1 overload

! ------------------------------------------------------------
! ROUTE PAR DEFAUT
! ------------------------------------------------------------
ip route 0.0.0.0 0.0.0.0 dhcp

! ------------------------------------------------------------
! SSH — Accès routeur uniquement depuis VLAN 10
! ------------------------------------------------------------
ip domain-name lan.local
ip ssh version 2
ip ssh bulk-mode 131072
crypto key generate rsa modulus 2048

ip access-list standard SSH_ADMIN_ONLY
 10 permit 192.168.10.0 0.0.0.255
 20 deny any log

line vty 0 4
 login local
 transport input ssh
 access-class SSH_ADMIN_ONLY in

line vty 5 14
 login local
 transport input ssh
 access-class SSH_ADMIN_ONLY in

! ------------------------------------------------------------
! HTTP — Interface d'admin web du routeur
! ------------------------------------------------------------
ip http server
ip http secure-server
ip http authentication local
ip http client source-interface GigabitEthernet0/0/1

end
write memory
```

---

## 2. Règles iptables par VM

> **Philosophie :** politique par défaut DROP sur INPUT et FORWARD, ACCEPT sur OUTPUT.
> Seuls les flux strictement nécessaires sont autorisés en entrée.

---

### VM Outil — 192.168.80.10 (Debian 13)

Rôle : contrôleur Ansible, accessible en SSH depuis VLAN 10 uniquement.

```bash
#!/bin/bash
# /etc/iptables/rules-outil.sh

# Vider les règles existantes
iptables -F
iptables -X
iptables -Z

# Politiques par défaut
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

# Loopback toujours autorisé
iptables -A INPUT -i lo -j ACCEPT

# Connexions établies/associées (réponses)
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# SSH depuis VLAN 10 (ADMIN) uniquement
iptables -A INPUT -p tcp --dport 22 -s 192.168.10.0/24 -m state --state NEW -j ACCEPT

# ICMP ping autorisé (diagnostic)
iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT

# Sauvegarder
iptables-save > /etc/iptables/rules.v4
```

---

### VM Web — 192.168.80.160 (Ubuntu Server)

Rôle : serveur Nginx. HTTP/HTTPS ouvert à tous. SSH depuis VLAN 30 et VM Outil.

```bash
#!/bin/bash
# /etc/iptables/rules-web.sh

iptables -F
iptables -X
iptables -Z

iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

# Loopback
iptables -A INPUT -i lo -j ACCEPT

# Connexions établies
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# HTTP/HTTPS depuis Internet (tout le monde)
iptables -A INPUT -p tcp --dport 80 -m state --state NEW -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -m state --state NEW -j ACCEPT

# SSH depuis VLAN 30 (DMZ) uniquement
iptables -A INPUT -p tcp --dport 22 -s 192.168.30.0/24 -m state --state NEW -j ACCEPT

# SSH depuis VM Outil (Ansible)
iptables -A INPUT -p tcp --dport 22 -s 192.168.80.10 -m state --state NEW -j ACCEPT

# ICMP
iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT

iptables-save > /etc/iptables/rules.v4
```

---

### VM BDD1 — 192.168.80.190 (Alpine)

Rôle : PostgreSQL. Accessible depuis VM Web (5432) et VM Outil (SSH). SSH depuis VLAN 20.

```bash
#!/bin/sh
# /etc/iptables/rules-bdd1.sh
# Alpine utilise /sbin/iptables

iptables -F
iptables -X
iptables -Z

iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

# Loopback
iptables -A INPUT -i lo -j ACCEPT

# Connexions établies
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# PostgreSQL depuis VM Web uniquement
iptables -A INPUT -p tcp --dport 5432 -s 192.168.80.160 -m state --state NEW -j ACCEPT

# SSH depuis VLAN 20 (INTERNE)
iptables -A INPUT -p tcp --dport 22 -s 192.168.20.0/24 -m state --state NEW -j ACCEPT

# SSH depuis VM Outil (Ansible)
iptables -A INPUT -p tcp --dport 22 -s 192.168.80.10 -m state --state NEW -j ACCEPT

# ICMP
iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT

# Sauvegarder sur Alpine
/etc/init.d/iptables save
```

---

### VM BDD2 — 192.168.80.170 (Alpine)

Identique à BDD1, changer uniquement le commentaire d'en-tête.

```bash
#!/bin/sh
# /etc/iptables/rules-bdd2.sh

iptables -F
iptables -X
iptables -Z

iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# PostgreSQL depuis VM Web uniquement
iptables -A INPUT -p tcp --dport 5432 -s 192.168.80.160 -m state --state NEW -j ACCEPT

# SSH depuis VLAN 20 (INTERNE)
iptables -A INPUT -p tcp --dport 22 -s 192.168.20.0/24 -m state --state NEW -j ACCEPT

# SSH depuis VM Outil (Ansible)
iptables -A INPUT -p tcp --dport 22 -s 192.168.80.10 -m state --state NEW -j ACCEPT

iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT

/etc/init.d/iptables save
```

---

### VM BDD3 — 192.168.80.180 (Alpine)

Identique à BDD1/BDD2.

```bash
#!/bin/sh
# /etc/iptables/rules-bdd3.sh

iptables -F
iptables -X
iptables -Z

iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# PostgreSQL depuis VM Web uniquement
iptables -A INPUT -p tcp --dport 5432 -s 192.168.80.160 -m state --state NEW -j ACCEPT

# SSH depuis VLAN 20 (INTERNE)
iptables -A INPUT -p tcp --dport 22 -s 192.168.20.0/24 -m state --state NEW -j ACCEPT

# SSH depuis VM Outil (Ansible)
iptables -A INPUT -p tcp --dport 22 -s 192.168.80.10 -m state --state NEW -j ACCEPT

iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT

/etc/init.d/iptables save
```

---

## 3. Notes importantes

### Appliquer les règles iptables au démarrage

**Debian/Ubuntu :**
```bash
apt install iptables-persistent
# Les règles dans /etc/iptables/rules.v4 sont chargées automatiquement
```

**Alpine :**
```bash
apk add iptables
rc-update add iptables
# /etc/init.d/iptables save  → sauvegarde dans /etc/iptables/rules-save
```

### Appliquer les règles à chaud (sans reboot)
```bash
chmod +x rules-xxx.sh
./rules-xxx.sh
```

### Vérifier les règles actives
```bash
iptables -L -v -n --line-numbers
```

### Résumé des flux autorisés

| Source | Destination | Port | Autorisé |
|---|---|---|---|
| VLAN 10 | VM Outil (80.10) | 22 (SSH) | ✅ |
| VLAN 10 | Hyperviseur (80.253) | 22 (SSH) | ✅ via routeur |
| VLAN 20 | BDD1/2/3 | 22 (SSH) | ✅ |
| VLAN 30 | VM Web (80.160) | 22 (SSH) | ✅ |
| Internet | VM Web (80.160) | 80/443 | ✅ |
| VM Outil | Toutes VMs | 22 (SSH) | ✅ |
| VM Web | BDD1/2/3 | 5432 (PG) | ✅ |
| Tout le reste | — | — | ❌ DROP |
