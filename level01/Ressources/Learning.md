/proc/<pid>/cmdline te donne la commande exacte avec laquelle le processus a été lancé — le binaire et tous ses arguments

ps aux affiche la liste de tous les processus en cours d'exécution sur la machine, avec leurs détails. C'est l'outil de base pour « voir ce qui tourne ». 

# Level01 — Notions & fiches techniques

Document de référence sur les concepts rencontrés à level01 : les processus,
le pseudo-système de fichiers `/proc`, les outils `ps`, `cat`, `tr`, et le
réflexe de reconnaissance « l'anomalie est la piste ». À garder comme fiche
transversale.

---

## 1. Qu'est-ce qu'un processus ?

Un **processus** = un **programme en cours d'exécution**. Quand un binaire est lancé,
le noyau en crée une instance vivante en mémoire, avec :

- un **PID** (Process ID) : son identifiant unique (ex. 691),
- un **propriétaire** (l'utilisateur sous lequel il tourne — ex. flag01),
- une **ligne de commande** (le binaire + ses arguments),
- un **environnement** (ses variables), des **fichiers ouverts**, etc.

**Daemon / service** : un processus qui tourne **en arrière-plan**, sans terminal, souvent
démarré au boot et relancé en continu (ex. un service `systemd`). C'est typiquement ce
qu'on chasse en CTF — un process privilégié qui tourne tout seul.

> Différence clé avec un simple fichier : un fichier est **statique** (on le lit/exécute),
> un process est **dynamique** (on peut lui parler, l'observer, exploiter comment il traite
> son entrée ou son environnement). Surface d'attaque plus riche.

---

## 2. Le pseudo-système de fichiers `/proc`

`/proc` n'est **pas** un vrai dossier sur le disque : c'est une **interface du noyau**
exposée *sous forme de fichiers*. Chaque processus vivant y a un répertoire
**`/proc/<pid>/`** qui reflète son état en temps réel.

Fichiers utiles dans `/proc/<pid>/` :

| Fichier | Contenu | Lisible par les autres ? |
|---|---|---|
| **`cmdline`** | la **ligne de commande** (binaire + arguments) | **oui** (world-readable) |
| `exe` | lien vers le **binaire** exécuté | oui |
| `cwd` | lien vers le **répertoire courant** du process | oui |
| `environ` | ses **variables d'environnement** | souvent **non** (owner-only) |
| `fd/` | les **fichiers ouverts** (descripteurs) | restreint |
| `status` | état, uids/gids, mémoire… | oui |
| `maps` | la cartographie mémoire | partiellement |

> C'est pour ça que `find -user flag01` renvoie des dizaines de lignes `/proc/691/...` :
> ce ne sont pas des fichiers sur disque, c'est l'image vivante d'un **process** de flag01.

---

## 3. La ligne de commande : `/proc/<pid>/cmdline`

Le fichier le plus précieux. Il contient le **binaire + tous les arguments** avec lesquels
le process a été lancé.

### Pourquoi c'est un réflexe
- **Toujours lisible** (world-readable), même pour un process d'un autre utilisateur.
- **Deux infos d'un coup** : quel binaire, et avec quels arguments.
- **Les arguments sont un spot de fuite classique** : mots de passe, tokens, clés passés
  en ligne de commande (`--password=...`) → visibles par tous. C'était la faille de level01.

### Le piège du format : séparateur NUL
Les arguments y sont séparés par des **octets NUL** (`\0`, code 0), **invisibles** à
l'affichage. Un `cat` brut colle donc tout :

```bash
cat /proc/691/cmdline
# /opt/snowcrash/level01/gargoylekooda7...   <- collé, illisible
```

D'où la commande complète :

```bash
cat /proc/691/cmdline | tr '\0' ' '; echo
# /opt/snowcrash/level01/gargoyle kooda7puivaav1idi4f57q8iq0   <- lisible
```

---

## 4. Outil : `tr` (translate)

`tr` **remplace** des caractères, un par un. Ce n'est **PAS** un outil de *trim* !

```bash
tr <source> <cible>        # remplace chaque caractère de <source> par celui de <cible>
```

- `tr 'a' 'b'` → remplace tous les `a` par des `b`
- `tr '\0' ' '` → remplace chaque **octet NUL** par un **espace** (notre cas)
- `tr 'a-z' 'A-Z'` → met en majuscules
- `tr -d '\n'` → l'option `-d` **supprime** (ici : enlève les retours à la ligne)
- `tr -s ' '` → l'option `-s` **compresse** les répétitions

> À retenir : `tr` **substitue** (ou supprime avec `-d`) des **caractères**, pas des mots.
> Il lit depuis l'entrée standard (stdin), d'où l'usage avec un pipe `|`.

---

## 5. Outil : `ps aux`

Affiche **tous les processus en cours** sur la machine.

```bash
ps aux
ps aux | grep flag01        # filtrer sur un utilisateur / un service
```

Options (syntaxe BSD, sans tiret) :
- **`a`** → process de **tous les utilisateurs**
- **`u`** → format détaillé **orienté utilisateur** (propriétaire, CPU, mémoire…)
- **`x`** → inclut les process **sans terminal** (daemons, services)

Colonnes clés :

| Colonne | Signification |
|---|---|
| USER | **qui** fait tourner le process |
| PID | identifiant (à passer à `/proc/<pid>/`) |
| TTY | terminal ; `?` = **daemon** (pas de terminal) |
| STAT | état (`R` tourne, `S` dort…) |
| COMMAND | **ligne de commande complète, arguments compris** ← la colonne en or |

> `ps aux` = la version « vue d'ensemble » de `cat /proc/<pid>/cmdline`. Même info
> (la COMMAND), mais pour **tous** les process d'un coup.

---

## 6. Rappel : `cat`, pipe `|`, `; echo`

- **`cat fichier`** → affiche le contenu brut d'un fichier.
- **`|` (pipe)** → envoie la sortie d'une commande dans l'entrée de la suivante
  (`cat ... | tr ...` : la sortie de `cat` devient l'entrée de `tr`).
- **`;`** → enchaîne deux commandes (la 2e s'exécute après la 1re, quoi qu'il arrive).
- **`echo`** (sans argument) → imprime une **ligne vide**. Ici, `; echo` sert juste à
  ajouter un retour à la ligne final (car `cmdline` n'en a pas), pour la propreté.

---

## 7. Contexte : services `systemd` & cgroups (aperçu)

La recon montrait aussi `.../gargoyle.service/...` dans `/sys/fs/cgroup`. Deux notions :

- **service systemd** : une unité gérée par `systemd` (le gestionnaire de services de Linux).
  Un nom **custom** (`gargoyle.service`) = installé exprès par l'auteur → piste probable.
  On l'inspecte avec `systemctl cat <service>` (voir le `User=`, l'`ExecStart=`).
- **cgroup** (control group) : mécanisme du noyau qui **regroupe et limite** des process
  (CPU, mémoire). L'arborescence `/sys/fs/cgroup/.../gargoyle.service/` indique juste que
  le process appartient à ce service — un indice de plus sur « qui l'a lancé ».

(Ces notions deviennent centrales aux niveaux à base de timers/services — level03, level04.)

---

## 8. La vulnérabilité de level01 : *secret in process arguments*

Le token était passé en **argument** du service `gargoyle`. Or les arguments de tout
process sont exposés dans `/proc/<pid>/cmdline` (world-readable) et via `ps` → **lisibles
par n'importe qui**.

> Règle réelle : ne **jamais** passer un secret en argument de commande
> (`mysql -pMOTDEPASSE`, `curl ...token`). Tout utilisateur de la machine peut le lire.
> À la place : variable d'environnement protégée, fichier à permissions restreintes, ou stdin.

---

## 9. Le réflexe transversal : « l'anomalie EST la piste »

En reconnaissance, sépare ce qui est **attendu** de ce qui est **inattendu** :

- **attendu** : ton home, les dotfiles, les binaires système habituels → du bruit.
- **inattendu** : ce que tu ne sais pas expliquer du premier coup → **ta piste**.

Ce qu'un auteur de challenge met **exprès** détonne par rapport à une machine normale.
Donc : **ne balaie jamais ce que tu ne comprends pas** — tire le fil.

À level01, l'avalanche `/proc/691` (inattendue pour une recherche de *fichiers*) était
précisément le signal : ta cible faisait tourner un process. C'est ce fil qui a mené au flag.

---

## Récap express

| Notion | En une phrase |
|---|---|
| Processus | un programme en cours d'exécution (PID, propriétaire, args…) |
| `/proc/<pid>/` | l'image vivante d'un process, exposée en fichiers par le noyau |
| `cmdline` | binaire + arguments (world-readable, args séparés par NUL) |
| `tr` | **remplace** des caractères (pas un trim ; `-d` pour supprimer) |
| `ps aux` | liste tous les process + leur ligne de commande |
| `/proc/<pid>/cmdline` vs `ps aux` | zoom ciblé vs vue d'ensemble |
| secret in args | un secret en argument = secret public |
| anomalie = piste | l'inexpliqué d'une recon est le fil à tirer |