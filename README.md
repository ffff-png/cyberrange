# Remote DNS Cache Poisoning — Scénario CyberRangeCZ (30 élèves, réponses variantes)

Scénario construit sur le même squelette que `library-junior-hacker`
(topologie Ansible + `training.json` pour la plateforme CyberRangeCZ /
KYPO CRP), mais l'attaque proposée est le **Remote DNS Cache Poisoning**
(Kaminsky attack) tel qu'enseigné dans le SEED Lab *"Remote DNS Cache
Poisoning Attack"*.

## Pourquoi les élèves auront chacun une réponse différente

CyberRangeCZ propose un mécanisme natif pour ça : **APG (Automatic
Problem Generation)**.

1. On déclare des variables aléatoires dans `variables.yml` (racine de
   la définition de sandbox, à côté de `topology.yml`).
2. Ansible les injecte dans la provision de chaque sandbox (donc
   chaque élève reçoit une zone DNS, une IP de redirection et un flag
   différents — pas juste un texte différent affiché, mais une vraie
   configuration différente sur sa VM).
3. Dans `training.json`, chaque niveau utilise `answer_variable_name`
   (au lieu de `answer` statique) + la case **Variant Answers**/
   `variant_sandboxes: true` activée.
4. Résultat : la plateforme vérifie la réponse de chaque élève contre
   **sa propre valeur générée**, donc copier la réponse d'un camarade
   ne fonctionne plus — même procédure, flag différent.

Il suffit de créer un **pool de 30 sandboxes** à partir de cette
définition : CyberRangeCZ régénère automatiquement les variables à
chaque instanciation de sandbox.

## Topologie (voir `topology.yml`)

| Host | Rôle | Image |
|---|---|---|
| `attacker` | Kali, lance l'attaque (script Scapy fourni) | kali |
| `attacker-ns` | Serveur DNS autoritaire **contrôlé par l'attaquant** pour une zone qu'il possède (`attacker_zone`) | debian-12 |
| `local-dns` | Résolveur DNS "victime" (BIND9, caching resolver) interrogé par le client | debian-12 |
| `client` | Poste utilisateur qui résout `target_domain` via `local-dns` | debian-12 |
| `rogue-server` | Faux site web vers lequel le DNS empoisonné redirige | debian-12 |
| `router` | Routeur entre les sous-réseaux | debian-12 |

Principe de l'attaque (classique SEED Lab) : `attacker` force
`local-dns` à interroger `attacker-ns` (en déclenchant une résolution
sur une sous-zone qu'il contrôle) puis court-circuite la vraie réponse
en injectant, avant qu'elle n'arrive, une fausse réponse DNS usurpant
l'autorité sur `target_domain` et pointant vers `spoofed_ip`
(l'IP de `rogue-server`). Une fois le cache de `local-dns` empoisonné,
`client` est redirigé vers `rogue-server` quand il résout
`target_domain`.

## Variables APG (`variables.yml`)

| Variable | Type | Usage |
|---|---|---|
| `attacker_zone` | text | nom de la zone que l'attaquant contrôle sur `attacker-ns` |
| `target_domain` | text | domaine victime à empoisonner |
| `spoofed_ip` | IP (restreinte au `target-switch`) | IP que le cache empoisonné doit renvoyer |
| `dns_src_port` | port | plage de ports source que le résolveur victime utilise (le rend "devinable", cœur de la vulnérabilité simulée) |
| `rogue_flag` | text | flag affiché sur la page du faux site, preuve que la redirection a fonctionné |

## Niveaux (`training.json`)

1. **Intro / Accès** — infos et connexion au sandbox (INFO/ACCESS, statiques).
2. **Reconnaissance** — `nmap` sur `target-switch`, identifier `local-dns`
   (port 53) et lire le nom du domaine à cibler sur la machine attaquante
   → flag = `target_domain` (variable).
3. **Prise de contrôle de la zone attaquante** — configurer/lire la zone
   que l'attaquant possède sur `attacker-ns` → flag = `attacker_zone`
   (variable).
4. **Lancement de l'attaque** — exécuter le script fourni
   (`/opt/attack/poison.py`, Scapy) qui force `local-dns` à interroger
   `attacker-ns` et course les réponses légitimes → flag = l'IP
   retournée par `dig target_domain @local-dns`, soit `spoofed_ip`
   (variable).
5. **Exploitation** — depuis `client`, visiter `target_domain` (déjà
   empoisonné) et récupérer le flag affiché par `rogue-server` → flag
   = `rogue_flag` (variable).
6. **Remédiation** — activer la randomisation des ports source +
   0x20 encoding sur `local-dns` (`named.conf.options`) → flag =
   commande utilisée pour recharger le service (statique, c'est une
   procédure identique pour tous, pas une donnée secrète).

## Déploiement

1. Importer ce dépôt comme **Sandbox Definition** dans CyberRangeCZ.
2. Créer un **Pool** de taille 30 sur cette définition.
3. Importer `training.json` comme **Training Definition** — vérifier
   que la case *Generate variant sandboxes* / `variant_sandboxes` est
   cochée (déjà `true` dans le fichier).
4. Créer une **Training Instance**, lui assigner le pool de 30, puis
   distribuer les tokens d'accès.

## À adapter avant usage réel

- Les rôles Ansible fournis (`attacker-ns`, `local-dns`, `rogue-server`)
  sont des **squelettes fonctionnels** (BIND9 + Flask/Nginx minimal) :
  testez-les sur votre instance CyberRangeCZ et ajustez les versions de
  paquets / chemins selon votre image Debian 12.
- Le script d'attaque Scapy (`files/poison.py`) illustre la logique
  (course de réponses forgées) ; en lab réel il faut le calibrer
  (nombre de paquets envoyés, fenêtre de port) pour que l'attaque
  réussisse de façon fiable dans vos conditions réseau.
- Ajoutez les rôles communautaires CyberRangeCZ que vous utilisez déjà
  (`user-access`, `chrony`, `sandbox-logging`, `hosts-aliases`) comme
  dans votre `library-junior-hacker` existant — ils ne sont pas
  dupliqués ici pour rester lisible.
