# Changelog — ADN QUIZZ

Récit des deux journées de travail des **1ᵉʳ et 2-3 août 2026**, du point de
départ à l'état actuel. `git show <hash>` donne le détail de chaque commit.

---

## Point de départ

Le programme fonctionnait (`ADNQU26P` / `ADNQ26WO`, commit `aca2f15`) mais le
texte de réponse était souvent illisible : la fonte est monochrome, chaque image
`.PI1` a sa propre palette, donc l'encre passait de lisible à invisible selon le
morceau.

---

## Journée 1 — rendre le texte lisible

### Les poids de plans étaient inversés — `bee167c`

`cptst` et `cpset` attribuaient le poids **8** au 1ᵉʳ plan et **1** au 4ᵉ, soit
l'inverse du matériel. Les deux fonctions étant cohérentes entre elles, la recopie
pixel-à-pixel marchait ; mais `recolor_font` leur passe un **vrai numéro de
registre**, qui se retrouvait peint bit-inversé.

L'encre d'origine étant la couleur 2, elle était repeinte en **4** au lieu de la
couleur demandée. Le correctif de contraste ne pouvait donc pas fonctionner.

### Les images sont au format STE, pas ST — `01c0a5b`

Découvert par l'analyse des palettes : `$0EC5`, `$0F77`… ont des composantes
supérieures à 7, impossibles en format ST 3 bits. Ce sont des images **STE 4096
couleurs**, où le 4ᵉ bit de chaque composante est le **LSB stocké en tête du
quartet**.

`test_color_contrast` lisait les quartets bruts : le classement par luminosité
était calculé sur des valeurs brouillées. **25 images sur 32** changent de couleur
choisie après correction.

### La fragmentation des glyphes — `b43f803`

`recolor_font` appelait `cptst`, une **FUNCTION** lisant le tableau global
`t160&()`, lui-même `DIM`ensionné dans `pmul160` et non dans `hello`.

Ce qui a permis de trancher : l'écran d'intro était épargné parce que sa palette
donne `ink_col&=2`, soit exactement la couleur d'origine de la fonte — la
recoloration y était un **no-op**. Les morceaux demandent la couleur 15, qui exige
d'allumer trois plans vides : le défaut n'y était plus masqué.

> La cause exacte reste une **hypothèse**. Ce qui a changé, c'est la suppression de
> l'appel de FUNCTION dans la boucle. Voir `todo.md` §4.

### Correctifs mémoire et documentation

- **`edff5b0`** — les tampons écran partaient de `log%` (arrondi à 256), les
  tampons fichiers de `mem_block%` (non arrondi). La fin de `pic%` pouvait déborder
  sur le **début de `sndhplay%`**, qui ne fait que 186 octets.
- **`bee167c`** — création de `README`, `AGENTS`, `CLAUDE`, `STRUCT`, `FEATURES`.
- **`54d0937`** — application de la règle **8.3** de GEMDOS à tous les fichiers.

---

## Journée 2 — fondus, effets, performances

### Les fondus n'étaient pas des fondus — `545877e`, `c8e1af6`

La boucle `ty&` montait le **rouge seul**, puis le bleu, puis le vert : l'écran
passait du noir au rouge, puis au magenta, et le vert — poids perceptuel 0,59 —
arrivait en dernier sur un rouge et un bleu déjà pleins.

Luminosité moyenne des 16 couleurs, trame par trame :

```
avant  0 0 1 1 2 2 2 3 3 3 3 3 3 3 3 3 3 4 4 5 5 6 6 7
                       plateau de 9 trames    puis surge
après  0 0 0 0 1 1 1 2 2 2 2 3 3 3 4 4 4 4 5 5 5 6 6 6 7
```

Trois défauts résiduels ensuite : la troncature laissait une composante faible à 0
jusqu'au pas 12 (« lent au début, rapide à la fin »), le registre 0 montait sur un
écran entièrement rempli de cette couleur (« flash sur certaines images » — mesuré
de 0,00 à 4,09 de luminosité selon l'image), et `cv&`/`cb&` n'étaient pas
initialisés dans deux branches.

> **Deux hypothèses fausses** écartées en chemin : le bornage d'une valeur négative,
> et le masque du rideau incomplet (simulé : 64/64 pixels). Consignées dans
> `FEATURES.md` pour qu'on ne les reprenne pas.

### La révélation : de 5 à 180 étapes — `57a5c8f`, `5942d84`, `4b8e1d4`

Contresens corrigé au passage : je présentais la lenteur comme le défaut, alors que
**la durée est la fonctionnalité** — la révélation doit s'étaler sur 2 minutes. Le
vrai défaut était qu'elle était **subie**.

| Étape | Résultat |
|---|---|
| `@pixdraw` par plans de bits | ~4500 `RC_COPY` VDI par niveau → 0 |
| Durée pilotée par `duree&` | chaque niveau dessiné vite, puis **tenu** |
| Paliers rectangulaires | la taille verticale n'est pas contrainte aux puissances de 2 → 9 niveaux |
| Révélation par bandes | 20 bandes de 10 lignes → **180 étapes**, une toutes les 0,67 s |
| Transformation au mot | 80 itérations par ligne au lieu de 160 |

### Quatre effets alternés — `259533d`, `c9e0ccb`, `c2b8f99`, `5a4c2ee`

`scrambling` cachait six défauts, dont un `EXIT IF` **commenté** dans la recherche
et `@tatouche` dans la boucle la plus interne (des centaines de milliers d'appels).
`random_pixels` n'avait **jamais tourné** et cumulait un `DIM done!(64000)` (le
motif documenté comme provoquant quatre bombes), une adresse fausse (`y&` jamais
affecté et compté en double) et un échantillonnage par rejet.

L'ancien dézoom `RC_COPY` avait été **supprimé par erreur** au lieu d'être conservé :
il est revenu sous le nom `dezoom.rc`, à côté des autres.

Tous partagent désormais l'interface `(duree&)` et finissent en **`dissolve.to.pic`** —
l'espace ne révèle jamais l'image d'un coup, elle se dévoile ligne par ligne sur
2,00 s exactement.

### Corrections finales — `1d61db9`, `bf89991`, `d5181f0`, `5d63a58`

- **Clignotement de `screen.opening`** : chaque itération n'écrivait que dans un
  tampon puis échangeait — les deux ne contenaient jamais la même chose. Plus un
  `BMOVE` de 320 octets là où une ligne en fait 160.
- **Ondulation du texte** : chaque lettre posée sur une sinusoïde, vague statique
  donc sans traînée.
- **`log%` effacé après `MALLOC`** : candidat principal pour l'écran brouillé au
  démarrage sur ST réel.
- **`dezoom.rc`** : ne travaillait pas dans le vide, il **attendait** 1,3 s d'un
  bloc après un balayage court. Attente répartie par rangée, 18 paliers.
- **Fondus portés à 1 s** par un second `VSYNC`.

---

## Historique des commits

| Date | Commit | Modification |
|---|---|---|
| 08-01 | `aca2f15` | Snapshot before debugging ADNQU26Q bus error |
| 08-01 | `bee167c` | Poids de plans dans cptst/cpset + documentation |
| 08-01 | `edff5b0` | Correctif mémoire (double origine des tampons) |
| 08-01 | `54d0937` | Règle 8.3 de GEMDOS |
| 08-01 | `01c0a5b` | Lecture de palette STE |
| 08-01 | `a27d0d9` | Description de FONT.INL |
| 08-01 | `b43f803` | Fragmentation des glyphes |
| 08-01 | `e5afca1` | Multiplication supprimée dans recolor_font, todo.md |
| 08-02 | `25294d1` | Fondus de palette, allègement de pixelisation |
| 08-02 | `545877e` | Rampe simultanée au lieu du balayage de canaux |
| 08-02 | `c8e1af6` | Flash résiduel, régularisation de la rampe |
| 08-02 | `57a5c8f` | Révélation pilotée en durée, rideau de texte global |
| 08-02 | `5942d84` | 72 étapes visibles |
| 08-02 | `4b8e1d4` | Vitesse de pixdraw ×2, 180 étapes |
| 08-02 | `0949e5a` | Audit des effets de révélation |
| 08-02 | `259533d` | Chantier 1 : scrambling piloté, alternance |
| 08-02 | `c9e0ccb` | dissolve.to.pic |
| 08-02 | `c2b8f99` | Chantier 2 : dézoom RC_COPY restauré |
| 08-02 | `5a4c2ee` | Chantier 3 : random_pixels réécrite, 4 effets |
| 08-03 | `3e07de0` | Échelle de dezoom.rc : neuf paliers utiles |
| 08-03 | `1d61db9` | Clignotement de screen.opening |
| 08-03 | `bf89991` | Ondulation du texte artiste / titre |
| 08-03 | `d5181f0` | log% effacé, attente de dezoom.rc répartie |
| 08-03 | `2fe5a69` | Points tranchés, GFX_WIP / OLD ignorés |
| 08-03 | `5d63a58` | changelog.md, fondus à 1 s, audit |

---

## Méthode retenue

- Développement sur le **seul `ADNQ27G.LST`** depuis `1d61db9`, un commit par étape.
- **Un changement à la fois**, diffé contre la dernière version qui tourne.
- Vérification **par simulation** avant livraison quand c'est possible : bornes
  mémoire, couverture d'écran, convergence, équivalence d'un refactoring.
- Les hypothèses non prouvées sont signalées comme telles, jamais présentées comme
  des faits.
