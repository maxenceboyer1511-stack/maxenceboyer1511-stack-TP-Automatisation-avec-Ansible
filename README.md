# 🛠️ TD/TP Ansible – Configuration d'infrastructure réseau

> **R4.ROM.09 DevOps** · IUT · F. BONNARDOT

Dans ce projet, j'automatise la configuration complète d'une infrastructure réseau d'entreprise à l'aide d'**Ansible** : un routeur Cisco CSR1000V et un serveur web Debian, le tout piloté depuis une VM de déploiement et versionné sur GitHub.

---

## 🗺️ Architecture de l'infrastructure

```
         Internet / Réseau IUT
                  │
            ┌─────┴─────┐
            │  WAN/DHCP │  ← GigabitEthernet1 (IP dynamique)
            └─────┬─────┘
                  │
           ┌──────┴──────┐
           │  CSR1000V   │  ← Routeur Cisco (NAT, DHCP, PAT)
           └──┬───┬───┬──┘
              │   │   │
    ┌─────────┘   │   └──────────┐
    │             │              │
LAN_ENTREPRISE  LAN_DMZ      LAN_ADMIN
192.168.2.0/24  192.168.3.0/24  192.168.4.0/24
    │             │              │
 PC clients    Serveur Web    VM Ansible
             (192.168.3.2)  (192.168.4.2)
```

| Interface        | Réseau           | Rôle                              |
|------------------|------------------|-----------------------------------|
| GigabitEthernet1 | WAN – DHCP IUT   | Accès Internet, IP publique       |
| GigabitEthernet2 | 192.168.2.1/24   | LAN Entreprise (clients + DHCP)   |
| GigabitEthernet3 | 192.168.3.1/24   | DMZ – Serveur Web                 |
| GigabitEthernet4 | 192.168.4.1/24   | LAN Admin – VM Ansible            |

---

## 📁 Structure du dépôt

```
td_ansible_infra/
├── install.sh                         # Installation des dépendances Ansible
└── ansible/
    ├── ansible.cfg                    # Config globale Ansible
    ├── hosts                          # Inventaire (routeurs + serveurs)
    ├── host_vars/
    │   ├── csr1000.yml                # Variables du routeur Cisco
    │   ├── web.yml                    # Variables du serveur web
    │   └── ansible_vm.yml             # Variables de la VM Ansible
    ├── playbook_lireconfigcsr.yml     # Lecture et sauvegarde de la config routeur
    ├── playbook_configurecsr.yml      # Configuration complète du routeur
    ├── playbook_webserver.yml         # Déploiement du serveur Apache
    └── files/
        └── site_web/
            └── index.html             # Mon site web déployé
```

---

## ⚙️ Installation

J'installe les dépendances sur la VM Ansible (Debian) en tant que super-utilisateur, puis je travaille ensuite uniquement avec mon compte utilisateur classique.

```bash
# En tant que root / sudo
sudo apt update
sudo apt install -y python3-pip python3-paramiko git
pip3 install ansible
```

> Le script `install.sh` à la racine du dépôt automatise ces 3 commandes.

---

## 🗂️ Configuration Ansible

### `ansible.cfg`

```ini
[defaults]
inventory = hosts
```

### `hosts` – Inventaire

```ini
[routeurs]
csr1000

[serveurs]
web
ansible_vm
```

### `host_vars/csr1000.yml` – Variables du routeur

```yaml
ansible_host: <IP_WAN_DU_CSR1000>
ansible_user: cisco
ansible_password: cisco
ansible_network_os: ios
ansible_connection: network_cli
```

### `host_vars/web.yml` et `host_vars/ansible_vm.yml`

```yaml
ansible_user: utilisateur
```

---

## 📋 Playbooks

### 1. Lecture de la configuration – `playbook_lireconfigcsr.yml`

Mon premier playbook : je lis et sauvegarde la configuration courante du routeur pour avoir une base de référence avant toute modification.

```bash
ansible-playbook ansible/playbook_lireconfigcsr.yml
```

---

### 2. Configuration du routeur – `playbook_configurecsr.yml`

C'est le playbook central du projet. Il configure tout le routeur de façon **idempotente** : si je le relance une deuxième fois sans rien changer, Ansible ne touche à rien (`changed=0`).

```bash
ansible-playbook ansible/playbook_configurecsr.yml
```

**Ce que fait ce playbook, étape par étape :**

#### 🔌 Interfaces réseau (`ios_l3_interfaces` + `ios_interfaces`)

| Interface        | Adresse IP         | Description         |
|------------------|--------------------|---------------------|
| GigabitEthernet1 | DHCP               | WAN – Accès IUT     |
| GigabitEthernet2 | 192.168.2.1/24     | LAN Entreprise      |
| GigabitEthernet3 | 192.168.3.1/24     | DMZ Serveur Web     |
| GigabitEthernet4 | 192.168.4.1/24     | LAN Admin Ansible   |

Chaque interface est activée (`no shutdown`) avec une description.

#### 📡 Serveur DHCP (`ios_config`)

Je configure un pool DHCP sur LAN_ENTREPRISE pour distribuer automatiquement des adresses aux postes clients. Comme il n'existe pas de module dédié au DHCP Cisco, j'utilise `ios_config`. J'ajoute aussi une tâche de suppression du pool existant en début de playbook pour éviter les erreurs en cas de réexécution.

#### 🌐 NAT/PAT dynamique (`ios_config` + `loop`)

Pour que les machines du LAN puissent accéder à Internet :

1. Je marque l'interface WAN en `ip nat outside`
2. J'utilise une boucle `loop` pour appliquer `ip nat inside` sur Gi2, Gi3 et Gi4 en une seule tâche
3. Je crée une ACL et active le NAT dynamique avec `overload`

```
access-list 1 permit 192.168.0.0 0.0.255.255
ip nat inside source list 1 interface GigabitEthernet1 overload
```

#### 🔀 PAT par port (redirection de ports)

Je redirige deux ports depuis l'IP publique du routeur vers mes serveurs internes :

| Port WAN | Destination interne | Service     |
|----------|---------------------|-------------|
| 80       | 192.168.3.2:80      | Serveur Web |
| 2222     | 192.168.4.2:22      | SSH Ansible |

---

### 3. Déploiement du serveur web – `playbook_webserver.yml`

J'installe et configure Apache2 sur la VM en DMZ (192.168.3.2), puis je déploie mon site.

```bash
ansible-playbook ansible/playbook_webserver.yml
```

**Étapes :**

1. Mise à jour du cache `apt`
2. Installation du paquet `apache2`
3. Copie de `files/site_web/` vers `/var/www/html/` (propriétaire `www-data`, droits `0600`)
4. Redémarrage et activation automatique du service Apache (`enabled: yes`, `state: restarted`)

> J'utilise `become: yes` dans ce playbook pour exécuter les tâches avec `sudo`.

---

## 🔗 Connexion à la VM Ansible depuis l'extérieur

Grâce au PAT configuré sur le routeur, je peux accéder à ma VM Ansible depuis mon poste via le port 2222 :

```bash
ssh utilisateur@<IP_WAN_CSR1000> -p 2222

# Ou via VS Code Remote SSH :
# utilisateur@<IP_WAN_CSR1000>:2222
```

---

## ✅ Tests de validation

```bash
# Vérifier que l'inventaire est bien lu
ansible-inventory --graph
ansible-inventory --list

# Test de connectivité vers tous les hôtes
ansible all -m ping

# Depuis une VM sur LAN_ENTREPRISE : vérifier que le NAT fonctionne
ping www.google.fr

# Depuis mon poste : tester le serveur web
curl http://<IP_WAN_CSR1000>:80
```

---

## 💡 Ce que j'ai retenu

- **L'idempotence** est la vraie force d'Ansible : je décris l'état souhaité, pas les commandes à exécuter. Si c'est déjà en place, Ansible ne fait rien.
- **`loop`** m'a permis d'éviter la répétition de tâches identiques sur plusieurs interfaces.
- **Les `host_vars`** rendent la configuration propre et facilement réutilisable d'une machine à l'autre.
- Contrairement à un simple script shell, Ansible vérifie l'état courant avant d'agir — ce qui le rend bien plus robuste pour de l'infra-as-code.

---

## 📚 Ressources

- [Collection Ansible – Cisco IOS](https://docs.ansible.com/ansible/latest/collections/cisco/ios/index.html)
- [Module `ios_l3_interfaces`](https://docs.ansible.com/ansible/latest/collections/cisco/ios/ios_l3_interfaces_module.html)
- [Module `ios_config`](https://docs.ansible.com/ansible/latest/collections/cisco/ios/ios_config_module.html)
- [Module `ansible.builtin.apt`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/apt_module.html)
- [Module `ansible.builtin.copy`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/copy_module.html)

---

*Projet réalisé dans le cadre du cours R4.ROM.09 DevOps – documents pédagogiques sous licence [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/).*
