# pfSense en haute disponibilité + Suricata

TP de Mastère 2 Cybersécurité (École IRIS Paris), réalisé sous VMware Workstation.

L'idée : mettre deux pare-feu pfSense en cluster pour que le réseau continue de fonctionner si l'un des deux tombe, et ajouter Suricata pour détecter les comportements suspects sur le réseau SOC.

Ce que j'ai mis en place :

- deux pfSense (un MASTER, un BACKUP) qui partagent 6 adresses virtuelles grâce à CARP
- la synchronisation des connexions en cours (pfsync) et de la configuration (XMLRPC) sur un lien dédié
- Suricata en mode IDS (détection seule) avec les règles ET Open
- un test de bascule : le MASTER passe en maintenance et le BACKUP reprend la main

> J'ai utilisé pfSense 2.7.2 alors que le TP vise la 2.8.1. Ça ne change rien au fonctionnement de CARP, pfsync et XMLRPC.

---

## Architecture

Chaque pfSense a 8 cartes réseau (em0 à em7). Sur chaque réseau interne, pfSense1 est en `.1`, pfSense2 en `.2`, et l'adresse virtuelle CARP partagée en `.3`. C'est cette adresse `.3` que les machines utilisent comme passerelle.

| Interface | Rôle | Réseau | VHID | pfSense1 | pfSense2 | VIP |
|---|---|---|---|---|---|---|
| WAN (em0) | Sortie internet (NAT VMware) | 192.168.237.0/24 | - | .50 | .51 | - |
| LAN (em1) | SOC | 192.168.10.0/26 | 10 | .1 | .2 | .3 |
| OPT1 (em2) | LANDOCKER | 192.168.60.0/28 | 20 | .1 | .2 | .3 |
| OPT2 (em3) | DMZ | 192.168.20.0/28 | 30 | .1 | .2 | .3 |
| OPT3 (em4) | AD | 192.168.30.0/26 | 40 | .1 | .2 | .3 |
| OPT4 (em5) | SYNC (lien dédié HA) | 172.16.1.0/24 | - | .2 | .3 | - |
| OPT5 (em6) | USERS | 192.168.50.0/24 | 50 | .1 | .2 | .3 |
| OPT6 (em7) | HONEYPOT | 192.168.40.0/28 | 60 | .1 | .2 | .3 |

- pfSense1 est le MASTER (skew 0), pfSense2 le BACKUP (skew 100)
- les réseaux internes sont en host-only (VMnet1 à 7), sans DHCP
- le client de test est une Ubuntu sur le réseau SOC, en 192.168.10.10/26
- domaine : `lab.lan`

```
                    Internet (NAT VMware)
                            │
              ┌─────────────┴─────────────┐
         WAN .50                      WAN .51
        ┌──────────┐   SYNC (OPT4)  ┌──────────┐
        │ pfSense1 │◄──────────────►│ pfSense2 │
        │  MASTER  │ pfsync+XMLRPC  │  BACKUP  │
        └────┬─────┘ 172.16.1.2/.3  └────┬─────┘
          .1 │        VIP CARP .3        │ .2
             └─────────────┬─────────────┘
                           │
     SOC · LANDOCKER · DMZ · AD · USERS · HONEYPOT
                           │
                 Client Ubuntu 192.168.10.10
```

---

## 1. Installation de pfSense1

Je commence par assigner les interfaces depuis la console. On ne crée pas de VLAN, chaque réseau a sa propre carte.

![Choix des VLAN](captures/01-console-assignation-vlan.png)

Premier souci : une réponse vide pendant l'assignation l'a arrêtée trop tôt, et seuls le WAN et le LAN ont été pris en compte.

![Assignation interrompue](captures/02-assignation-interrompue.png)

En recommençant, les 8 interfaces sont bien assignées :

![Assignation des 8 interfaces](captures/03-assignation-8-interfaces.png)

Puis j'attribue les adresses IP de chaque interface (option 2 de la console) :

![Adressage de pfSense1](captures/04-pfsense1-adressage.png)

## 2. Client de test et assistant de configuration

Sur l'Ubuntu, je pose une IP dans le réseau SOC et je vérifie que pfSense répond :

![Client - IP temporaire](captures/05-client-ip-temporaire.png)

J'accède ensuite à l'interface web depuis le client et je déroule l'assistant : nom d'hôte, domaine `lab.lan`, DNS, WAN en IP statique.

![Connexion](captures/06-webgui-login.png)
![Assistant - informations générales](captures/07-wizard-general.png)
![Assistant - WAN statique](captures/08-wizard-wan-static.png)
![Assistant terminé](captures/09-wizard-termine.png)

Le tableau de bord confirme la version 2.7.2 et les 8 interfaces actives :

![Tableau de bord pfSense1](captures/10-dashboard-pfsense1.png)
![Interfaces et mémoire](captures/11-dashboard-interfaces-memoire.png)

On voit déjà sur cette capture que la mémoire est très juste : 74 % de 193 Mo. J'y reviens plus bas.

## 3. Suricata en mode IDS

Installation du paquet depuis System > Package Manager :

![Installation de Suricata](captures/12-installation-suricata.png)

J'active les règles gratuites ET Open (Emerging Threats) et je lance la mise à jour :

![Règles ET Open](captures/13-suricata-regles-etopen.png)
![Mise à jour des règles](captures/15-suricata-maj-regles-1.png)
![Mise à jour des règles](captures/16-suricata-maj-regles-2.png)

Suricata surveille l'interface LAN (réseau SOC). Le blocage est désactivé : on est en IDS, il détecte et alerte mais ne bloque rien.

![Interface LAN](captures/14-interface-lan.png)
![Instance Suricata sur le LAN](captures/17-suricata-instance-lan.png)

Côté client, je rends l'IP permanente avec un profil `nmcli` (l'IP posée avec `ip addr add` disparaissait), puis je scanne le pare-feu avec nmap. Seuls les ports 80 et 443 de l'interface web sont ouverts.

![Profil nmcli et nmap](captures/18-client-nmcli-nmap.png)

Les premières alertes remontent. Ce sont des alertes de décodage (`QUIC failed decrypt`, `STREAM excessive retransmissions`) générées par la navigation normale du client, pas des attaques.

![Alertes Suricata](captures/19-suricata-alertes.png)

## 4. Mise en place de pfSense2

pfSense2 est un clone de pfSense1, avec les adresses en `.2` et `172.16.1.3` sur le lien SYNC :

![Adressage de pfSense2](captures/21-pfsense2-adressage.png)
![Tableau de bord pfSense2](captures/22-dashboard-pfsense2.png)

Le client joint bien les deux nœuds :

![Ping des deux nœuds](captures/25-client-ping-deux-noeuds.png)

## 5. Lien de synchronisation (OPT4)

Pour que les deux pare-feu se parlent, il faut une règle qui autorise le trafic sur OPT4, sur chaque nœud :

![Règle OPT4](captures/26-regle-opt4-sync.png)

Le ping SYNC passe dans les deux sens :

![Ping vers pfSense2](captures/23-ping-sync-vers-pfsense2.png)
![Ping vers pfSense1](captures/24-ping-sync-vers-pfsense1.png)
![Ping depuis le shell de pfSense1](captures/27-ping-sync-shell-pfsense1.png)
![Ping depuis le shell de pfSense2](captures/28-ping-sync-shell-pfsense2.png)

## 6. pfsync : synchronisation des états

pfsync copie la table des connexions en cours d'un nœud à l'autre. Si le MASTER tombe, le BACKUP connaît déjà les connexions ouvertes et elles ne sont pas coupées.

![Configuration pfsync](captures/29-pfsync-pfsense2.png)

Preuve que ça marche : les connexions du client, visibles sur pfSense1…

![États sur pfSense1](captures/30-etats-pfsense1.png)

…apparaissent aussi sur pfSense2, avec 0 paquet, puisqu'elles ont été copiées et pas traitées par lui :

![États synchronisés sur pfSense2](captures/31-etats-pfsense2-synchronises.png)

## 7. XMLRPC : synchronisation de la configuration

XMLRPC est configuré uniquement sur le MASTER : il pousse sa configuration vers le BACKUP (règles de pare-feu, adresses virtuelles, etc.).

![Configuration XMLRPC](captures/32-xmlrpc-pfsense1.png)

Test : je crée une règle `TEST-XMLRPC-HA` sur pfSense1…

![Règle de test sur pfSense1](captures/33-regle-test-xmlrpc-pfsense1.png)

…et elle apparaît toute seule sur pfSense2 :

![Règle de test sur pfSense2](captures/34-regle-test-xmlrpc-pfsense2.png)

## 8. Adresses virtuelles CARP

Une adresse virtuelle par réseau interne, chacune avec son propre VHID :

![VIP CARP](captures/35-vip-carp.png)

Le client utilise maintenant la VIP `192.168.10.3` comme passerelle, et internet répond :

![Client via la VIP](captures/36-client-passerelle-vip.png)

## 9. Test de bascule

Je passe pfSense1 en mode maintenance CARP. Il bascule en BACKUP sur les 6 VIP :

![pfSense1 en BACKUP](captures/37-carp-pfsense1-backup.png)

Et pfSense2 reprend le rôle de MASTER :

![pfSense2 en MASTER](captures/38-carp-pfsense2-master.png)

Pendant le test, le client tournait un `ping -i 0.2 8.8.8.8`. Pour comparer, sans aucune panne j'avais déjà 66 réponses sur 68 envois, à cause de pertes isolées dues au NAT VMware.

---

## Problèmes rencontrés

**Assignation des interfaces interrompue.** Une réponse vide a arrêté l'assignation après le LAN. Il a fallu la relancer et saisir les 8 interfaces.

**IP du client qui disparaît.** L'adresse posée avec `ip addr add` était écrasée par NetworkManager. Solution : un profil `nmcli` permanent.

**Ping SYNC qui échoue.** Trois causes à la suite : règle absente sur pfSense1, protocole TCP ou ICMP au lieu de « Any » (pfsync utilise son propre protocole, le n°240), et oubli du bouton « Apply Changes ».

**La vraie cause racine : la RAM.** Les VM n'avaient que 256 Mo. pfSense n'arrivait plus à charger ses règles (`Cannot allocate memory`, `pf rules not loaded`), ce qui expliquait plusieurs comportements bizarres. Passage à 2048 Mo par VM.

![VM à 256 Mo](captures/20-vmware-256mo.png)

**Clavier QWERTY sur la console.** Impossible de taper la barre `|`. J'ai contourné le problème avec Diagnostics > Command Prompt dans l'interface web.

---

## Ce que j'en retiens

- CARP ne fait que déplacer une adresse IP. Sans pfsync, les connexions seraient coupées à chaque bascule, et sans XMLRPC, les deux pare-feu finiraient par avoir des règles différentes.
- Un lien de synchronisation dédié, c'est plus propre et plus sûr que de faire passer pfsync par le LAN.
- La config de Suricata n'est pas synchronisée par XMLRPC. Ici, elle est identique sur les deux nœuds uniquement parce que j'ai cloné la VM.
- Quand plusieurs problèmes arrivent en même temps, il vaut mieux chercher la cause commune. Ici, c'était la mémoire.

## Pistes d'amélioration

- passer en pfSense 2.8.1
- changer le mot de passe admin par défaut
- activer la synchronisation de Suricata entre les deux nœuds
- tester une vraie panne (VM éteinte) en plus du mode maintenance

---

Djé Yoboué · [Portfolio](https://djeyoboue44-glitch.github.io) · [GitHub](https://github.com/djeyoboue44-glitch)
