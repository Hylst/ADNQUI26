# Instructions pour les agents — ADN QUIZZ

Ce fichier prime sur toute habitude générale. Lire entièrement avant de modifier
une ligne de code.

## Règles de méthode

1. **Un seul changement à la fois**, systématiquement diffé contre la dernière
   version qui tourne (`ADNQ26WO.LST`). Le cycle de test est lent et coûteux pour
   l'utilisateur : un test qui valide deux changements ne valide rien.
2. **Ne jamais écraser `ADNQ26WO.*`** — c'est le point de retour.
   Faire une copie de sauvegarde avant d'éditer `ADNQ26Q.LST`.
3. **Vérifier avant d'affirmer.** Plusieurs erreurs passées viennent d'hypothèses
   non vérifiées sur `RC_COPY`, la table VDI ou `RESERVE`. Sources §Documentation.
4. **Le manuel donne des règles générales que ce programme contredit
   délibérément** (cas de `RESERVE`). Lire le code environnant et ses commentaires
   avant de « corriger » ce qui ressemble à une négligence.
5. **Approche pessimiste sur l'état des fichiers** : l'utilisateur édite en
   parallèle dans l'éditeur GFA. Relire le fichier avant d'éditer, ne pas se fier
   à une lecture antérieure dans la même session.

## Noms de fichiers — 8.3 obligatoire

GEMDOS impose **8 caractères maximum, extension 3 maximum, un seul point**.
Le dossier étant monté comme disque par l'émulateur, la règle vaut pour **tout**
fichier qu'on y crée, y compris la documentation.

```
ADNQ26Q.LST    ✔  7 + 3
STRUCT.md      ✔  6 + 2
STRUCTURE.md   ✗  9 caracteres
ADNQ26Q.LST.avant-planfix   ✗  deux points, base et extension trop longues
```

Seule exception tolérée : `.gitignore`, exigé par git et jamais lu par le programme.

**Ne pas créer de fichier de sauvegarde suffixé** (`*.avant-*`, `*.old`…) :
c'est git qui tient ce rôle, et ces noms violent la règle.

## Contraintes d'édition du `.LST`

- Encodage **UTF-8, fins de ligne CRLF**. Le fichier contient déjà 41 octets > 127,
  dont 9 caractères de remplacement `U+FFFD` (dégâts antérieurs, ne pas aggraver).
- **N'écrire que de l'ASCII** dans le code et les commentaires ajoutés : pas
  d'accents, pas de tiret cadratin, pas d'apostrophe typographique. L'éditeur GFA
  sur l'Atari ne lit pas l'UTF-8 multi-octets.
- Ne pas allonger inutilement les lignes (un « line too long » non expliqué a déjà
  été rencontré au Merge).
- Après édition, **vérifier le diff complet** avant de rendre la main.

## Pièges confirmés — ne pas les refaire

| Piège | Réalité |
|---|---|
| `RESERVE` désactivé en compilé « comme dit le manuel » | **Casse tout.** `$m 400000` donne 400 Ko au programme, `RESERVE 400000` libère le surplus vers GEMDOS, et le `MALLOC` (~488 Ko) ne peut venir **que** de cette libération. Sans lui : `mem_block%<=0` → `EDIT` → écran noir. Erreur commise deux fois. |
| `COLOR 15` pour le masque du rideau | **Faux.** `COLOR` attend un index **VDI**. VDI 15 = registre matériel 13 = `1101` → un plan perdu. `COLOR 1` = registre 15 = `1111` → correct. |
| Passer un numéro de registre à `COLOR`/`DEFFILL` | Toujours convertir : `COLOR vdicol|(registre)`. |
| Appeler une `FUNCTION` qui touche un tableau global depuis une boucle | Provoque `ERR 15 Array not dimensioned` par intermittence (`cptst` dans `recolor_font`). Contournement : recopier la logique **dans** la PROCEDURE. |
| `ADNQU26P.LST` serait un export périmé | **Faux**, MD5 identique à `ADNQ26WO.LST`. |

## Mécanismes vérifiés

### Composition du texte — deux `RC_COPY`

```gfa
RC_COPY fmask%,m.appear&,0,8,8 TO log%,x,y          ! mode 3 par defaut = replace
RC_COPY font%,...,cw,ch TO log%,x,y,1               ! mode 1 = s AND d
```

Modes `RC_COPY` (opération logique bit à bit sur les 4 plans) : `0` tout à 0,
**`1` = s AND d**, **`3` = replace (défaut)**, `6` XOR, `7` transparent (s OR d),
`13` transparent inverse, `15` tout à 1.

**Conséquence majeure** : le masque est tracé en `COLOR 1` (= registre 15 = 4 plans
à 1). Là où il est allumé, le AND préserve la fonte ; là où il est éteint, le
résultat est **0**. Donc **le fond de chaque cellule 8×8 est toujours le registre
matériel 0** — c'est avec lui que l'encre doit contraster, pas avec l'image.

### Index VDI ≠ registre matériel

```
index VDI   : 0  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15
registre HW : 0 15  1  2  4  6  3  5  7  8  9 10 12 14 11 13
```

Le programme embarque la table inverse `vdicol|()` :
`DATA 0,2,3,6,4,7,5,8,9,10,11,14,12,15,13,1`, soit `vdicol|(registre) = index VDI`.
Contrôle : `vdicol|(15) = 1` ✔

### Plans de bits — 320×200×16

4 plans entrelacés, 4 mots de 16 bits consécutifs pour 16 pixels (8 octets).
**Le 1er mot en mémoire est le bit de poids FAIBLE (LSB)**, le 4ᵉ le poids fort :

```
couleur = (b3 × 8) + (b2 × 4) + (b1 × 2) + (b0 × 1)
```

`cptst` / `cpset` respectent cette convention **depuis la correction du 2026-08-01**
(elles étaient inversées, ce qui faisait peindre `recolor_font` en bit-inversé).

### Format de palette

| Machine | Encodage du mot | Remarque |
|---|---|---|
| ST | `.....RRR .GGG .BBB` | 3 bits par composante |
| STe | `....rRRR gGGGbBBB` | 4 bits, **le bit supplémentaire est le LSB mais stocké en tête du quartet** |

Les images `.PI1` portent une palette **format ST** : lire le quartet directement
donne 0–7, ce qui est correct. Sur une palette **STe** en revanche, lire le quartet
brut donne une valeur bit-brouillée — attention si le programme évolue vers 4096 couleurs.

## Documentation à consulter

**Base interrogeable** (41 ouvrages indexés, voir `CLAUDE.md` pour les commandes) :
`D:\0CODE\ClaudeCode\atari\` — `python tools/kb.py lookup <symbole> --full`.

**Manuels bruts**, décisifs à plusieurs reprises, dans `D:\0CODE\OpenCode\Pixel_Bitmap_Gen\` :

- `GFABasic.HYP.html` — manuel GFA BASIC complet, le plus fiable
- `GFAXPERT2.txt` — *Your Second GFA-BASIC 3.0 Manual* (Han Kempen), contient la table VDI
- `gfamanual.txt` — manuel de référence

## Historique

`continue.md` est le journal détaillé de la session de débogage du 2026-08-01
(hypothèses fausses écartées, audit ligne à ligne). Le consulter avant de rouvrir
un sujet qui semble déjà traité.
