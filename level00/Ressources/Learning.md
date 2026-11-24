# Level00 — Notions & fiches techniques

Document de référence sur les concepts rencontrés à level00 : permissions,
bit SUID, anatomie d'un binaire, assembleur vs code machine, et les outils
(`strings`, `objdump`, `find`). À garder comme fiche transversale.

---

## 1. Les permissions Unix

Chaque fichier a des permissions pour **3 catégories** : propriétaire (*user*),
groupe (*group*), autres (*other*).

```
-rwxr-x---
│└┬┘└┬┘└┬┘
│ │  │  └── other : ---      (aucun droit)
│ │  └───── group : r-x      (lecture + exécution)
│ └──────── user  : rwx      (lecture + écriture + exécution)
└────────── type  : - (fichier), d (dossier), l (lien)
```

Trois droits de base :
- **r** (read / 4) — lire le contenu (ou *lister* un dossier)
- **w** (write / 2) — modifier (ou créer/supprimer dans un dossier)
- **x** (execute / 1) — exécuter (ou *traverser* un dossier)

Notation octale : `rwx` = 7, `r-x` = 5, `rw-` = 6. Ex. `chmod 644` = `rw-r--r--`.

> Nuance dossier : pour un **dossier**, `x` = droit de *traverser* (accéder à un
> fichier dont on connaît le nom), `r` = droit de *lister* le contenu. C'est pour ça
> qu'on peut parfois `cat /dir/fichier` sans pouvoir `ls /dir`.

---

## 2. Le bit SUID (Set User ID)

Le point central de level00.

Normalement, un programme s'exécute avec les droits de **celui qui le lance**.
Le bit **SUID** change ça : le programme s'exécute avec les droits de son **propriétaire**.

Il se repère par un **`s`** à la place du `x` du propriétaire :

```
-r-sr-x--- 1 flag00 level00 ... hiro
   ↑
   s = SUID : exécuté avec les droits de flag00, quel que soit le lanceur
```

- SUID (`u+s`, octal `4000`) → prend l'identité du **propriétaire**
- SGID (`g+s`, octal `2000`) → prend l'identité du **groupe**

**Pourquoi ça existe** : certains programmes légitimes en ont besoin (ex. `passwd` doit
écrire dans `/etc/shadow`, qui n'appartient qu'à root → `passwd` est SUID root).

**Pourquoi c'est dangereux** : un binaire SUID mal écrit = une porte d'escalade de
privilèges. C'est *le* mécanisme de base de SnowCrash.

### uid réel vs uid effectif
- **uid réel (ruid)** : qui tu es vraiment (celui qui a lancé).
- **uid effectif (euid)** : l'identité utilisée pour les **contrôles d'accès**.

Un SUID met l'euid = propriétaire. `id` affiche les deux (`uid=` et, s'ils diffèrent,
`euid=`).

---

## 3. Anatomie d'un binaire (ELF)

Sous Linux, un exécutable est au format **ELF** (Executable and Linkable Format).
Il est découpé en **sections** nommées, chacune avec un rôle :

| Section | Contenu |
|---|---|
| `.text` | le **code machine** (les instructions exécutables) |
| `.rodata` | données constantes en lecture seule (chaînes, tables) |
| `.data` | variables globales initialisées |
| `.bss` | variables globales non initialisées (réservées, pas stockées) |
| `.symtab` | table des symboles (noms de fonctions/variables) — absente si *stripped* |
| `.flag_data` | **section custom** (créée par l'auteur du challenge) → le flag y était rangé |

**stripped vs not stripped** : un binaire *stripped* a eu sa table de symboles retirée
(`main`, noms de fonctions… disparaissent → reverse plus dur). `hiro` était
**not stripped**, donc on a vu tous les noms de fonctions dans `strings`.

---

## 4. Code machine vs assembleur

- **Code machine** : les **octets bruts** exécutés par le CPU. C'est ce qui est
  *réellement* dans le binaire (section `.text`). Ex : `55`, `48 89 e5`.
- **Assembleur** : la **représentation textuelle lisible** de ces octets, avec des
  mnémoniques. Ex : `55` → `push rbp` ; `48 89 e5` → `mov rbp, rsp`.

Correspondance **1:1** (1 instruction asm ↔ 1 instruction machine), mais le binaire ne
stocke **que les octets**. L'assembleur est reconstruit par un désassembleur.

### Chaîne de compilation
```
code C  ──(compilateur)──►  assembleur (.s)  ──(as)──►  code machine (.o)  ──(linker)──►  exécutable
```
Le texte assembleur n'est qu'une étape intermédiaire ; il a disparu du binaire final.

### Reverse = chemin inverse
```
binaire (octets)  ──(objdump -d / gdb / Ghidra)──►  assembleur  ──(décompilateur)──►  pseudo-C
```

---

## 5. Outil : `strings`

Extrait toutes les **suites de caractères imprimables** d'un fichier (≥ 4 par défaut).

```bash
strings fichier            # toutes les chaînes
strings -n 8 fichier       # seulement celles d'au moins 8 caractères (moins de bruit)
strings -t x fichier       # préfixe chaque chaîne de son offset en hexa
strings -a fichier         # scanne tout le fichier, pas que les sections de données
```

- **Statique et sans risque** : on lit, on n'exécute pas.
- Révèle : messages, chemins, noms de commandes appelés, noms de fonctions, et
  parfois des **secrets codés en dur** (le cas de level00).
- **Limite** : ne montre ni la logique, ni les chaînes chiffrées/calculées au runtime.

---

## 6. Outil : `objdump`

Inspecte la structure interne d'un ELF.

```bash
objdump -d fichier                 # désassemble la section .text (code -> assembleur)
objdump -s fichier                 # dump hex+ASCII du contenu brut des sections
objdump -s -j .flag_data fichier   # idem mais UNIQUEMENT la section .flag_data
objdump -t fichier                 # table des symboles
objdump -h fichier                 # liste des sections (headers)
```

- **`-d`** (disassemble) : traduit le code machine en assembleur lisible.
- **`-s`** (full contents) : affiche les octets bruts (hex) + leur ASCII à droite.
- **`-j <section>`** : restreint à une seule section (sinon `-s` dumpe tout).

C'est `objdump -s -j .flag_data` qui a isolé **chirurgicalement** le flag de level00,
sans le bruit de `strings`.

---

## 7. Outil : `find` (recherche de cibles)

```bash
find / -user flag00 2>/dev/null        # fichiers appartenant à flag00
find / -group burbclave 2>/dev/null    # fichiers appartenant à un groupe
find / -perm -4000 -type f 2>/dev/null  # tous les binaires SUID
```

- `-perm -4000` : le `-` signifie « **au moins** ces bits » → tout ce qui a le bit SUID.
- `2>/dev/null` : jette les erreurs « Permission denied » pour ne garder que les résultats utiles.
- `-type f` : seulement les fichiers (pas les dossiers).

---

## 8. La vulnérabilité de level00 : *hardcoded secret*

Un secret (ici le flag) **compilé en dur** dans un binaire. Dès que quelqu'un peut
**lire** le binaire (ici via le groupe `level00`, grâce au bit `r` de groupe),
il l'extrait avec `strings`/`objdump` — **sans jamais exécuter** le programme.

> Règle réelle : ne **jamais** stocker un secret dans un exécutable. Un binaire distribué
> est lisible → le secret est public. On utilise des variables d'env protégées, des
> fichiers à permissions restreintes (600), ou un coffre (vault).

---

## 9. Le piège de level00 : ne pas croire l'affichage

`hiro` simulait un faux shell flag00 (faux prompt, faux `id`, faux `cat`). Leçon
transversale :

- Un programme peut **afficher n'importe quoi** — un prompt ou un `id` affiché ne
  prouve rien.
- La seule identité fiable vient du **noyau**, via un binaire de confiance appelé par
  **chemin absolu** : `/usr/bin/id` (que l'auteur ne peut pas remplacer), pas `id`
  (qui dépend du `PATH`, potentiellement trafiqué).

```bash
/usr/bin/id        # la vérité
echo $PATH         # vérifier qu'aucun dossier piégé n'est en tête
```

---

## Récap express

| Notion | En une phrase |
|---|---|
| Permissions | 3 catégories (user/group/other) × 3 droits (rwx) |
| SUID (`s`) | s'exécute avec les droits du **propriétaire** |
| ELF / sections | binaire découpé en zones nommées (`.text`, `.rodata`, `.flag_data`…) |
| Code machine | octets exécutés par le CPU (ce qui est *dans* le binaire) |
| Assembleur | représentation lisible 1:1 du code machine (reconstruite) |
| `strings` | extrait le texte brut d'un fichier |
| `objdump -d` | désassemble le code |
| `objdump -s -j` | dump brut d'une section précise |
| hardcoded secret | secret dans le binaire = secret public |