# TODO — ADN QUIZZ

> Fichier de travail : **`ADNQ27G.LST`** uniquement, avec un commit git par étape.
> Historique et décisions passées : `changelog.md`. État des fonctionnalités :
> `FEATURES.md`. Méthode et pièges : `AGENTS.md`.

---

## Plan de bataille

### En cours / immédiat

| # | Chantier | Où | État |
|---|---|---|---|
| 1 | **Ondulation animée** du texte jusqu'à la touche, avec **restauration ciblée du fond** | §11 | à faire |
| 2 | **Feu d'artifice** en remplacement de l'image finale digitalisée | §12 | à concevoir |

### Ensuite

| # | Chantier | Où |
|---|---|---|
| 3 | Choix de l'encre par **rareté × contraste** (rend le cyclage de palette viable) | §13 |
| 4 | **Contour** autour des glyphes au lieu du bandeau noir → texte sur l'image | §10.4 |
| 5 | Scrolltext d'intro avec `2p_16_22.pi1` | §12.3 |
| 6 | Nettoyage : `chk$`, `ALERT`, code mort | §10.2, §10.3 |
| 7 | Passer les procédures aux `LOCAL` | §10.1 |

### Bloqué sur de l'assembleur — pour plus tard

| Sujet | Pourquoi | Où |
|---|---|---|
| Raster / Timer B (barres, scroll matériel STE) | GFA ne peut pas produire un `RTE` ni acquitter le MFP | §8.3, §8.4 |
| Fondu du volume musical | le lecteur SNDH réécrit les registres YM chaque trame sous interruption | §9.2 |

Le projet a déjà le motif pour ça : `DATA\SNDHPLAY.INL` est une routine machine
chargée par `BLOAD`. Une routine raster suivrait le même chemin.

---


## 1. Effet de transition (`pixelisation`, l. 403)

L'effet fonctionne mais il est très lent. Voici ce que le code fait aujourd'hui,
et ce qui coûte cher — par ordre de gain décroissant.

### 1.1 Le coût réel : un `RC_COPY` par bloc de 4×4 — ✅ FAIT (`ADNQ26Z`)

> ⚠️ **Ma formulation initiale était fausse.** Je présentais la lenteur comme le
> défaut à corriger. En réalité **la durée est la fonctionnalité** : la révélation
> doit s'étaler jusqu'à 2 minutes pour laisser deviner les joueurs. Ce qu'il
> fallait corriger, c'est que la durée était subie (elle dépendait de la lenteur
> du dessin) au lieu d'être pilotée. `ADNQ26Z` sépare les deux : `@pixdraw`
> dessine vite, `@pixelisation(duree&)` tient chaque niveau le temps voulu.


```gfa
FOR j&=0 TO bloc_h&-1 STEP 4
  FOR i&=0 TO bloc_l&-1 STEP 4
    RC_COPY pic%,x&,y&,4,4 TO phy%,x&+i&,y&+j&
```

Avec `taille&=32`, chaque bloc déclenche `(32/4)² = 64` appels `RC_COPY`, et il y a
`(320/32)×(200/32) ≈ 70` blocs — soit **~4500 `RC_COPY` par cycle**, chacun passant
par le VDI. C'est le poste de dépense principal, très loin devant tout le reste.

**Piste : travailler au niveau des plans de bits plutôt qu'en `RC_COPY`.**

Pour une taille de bloc de 16 pixels, la réplication horizontale est quasi gratuite —
un groupe de 16 pixels tient dans un mot par plan :

```gfa
' pour chaque mot de chaque plan : si le pixel de gauche est allume,
' tout le mot s'allume ; sinon tout s'eteint
IF (CARD{src%} AND &H8000)
  CARD{dst%}=&HFFFF
ELSE
  CARD{dst%}=0
ENDIF
```

Pour 8, 4 ou 2 pixels, le même principe s'applique par demi-mot / quart de mot.
Le plus simple est une **table de 256 entrées par taille de bloc** : `octet source →
octet pixelisé`, précalculée une fois au démarrage. L'effet devient alors une
boucle de lectures de table sans une seule multiplication.

**Puis la réplication verticale est encore moins chère** : une fois la première
ligne d'une rangée de blocs construite, un `BMOVE` de 160 octets la recopie sur les
`taille&-1` lignes suivantes. Pas de calcul par pixel du tout.

### 1.2 `@tatouche` appelé dans la boucle la plus interne — ✅ FAIT (`ADNQ26V`)

`@tatouche` (test clavier) est appelé à **quatre niveaux d'imbrication**, jusque
dans la boucle `i&`. À ~4500 itérations par cycle, l'appel de procédure seul pèse
plus que le travail utile.

Le remonter **une fois par ligne de blocs** suffit largement : la réactivité au
clavier reste imperceptiblement différente pour l'utilisateur.

**Appliqué** : les appels des boucles `i&` et `j&` sont retirés ; il subsiste au
niveau de la boucle `x&`. Reste le poste principal, §1.1, non traité.

### 1.3 Le calcul de `taille&` fait une puissance et deux divisions

```gfa
taille&=2^(5-cycle&/(maxcycle&/5))
```

`^` passe par le flottant, et il y a deux divisions — le tout à chaque cycle.
Une table `DATA` rend la progression explicite, réglable à l'œil, et gratuite :

```gfa
RESTORE tailles
FOR c&=1 TO 16
  READ pix.size&(c&)
NEXT c&
tailles:
DATA 32,32,16,16,16,8,8,8,4,4,4,2,2,2,1,1
```

Bénéfice secondaire : la courbe de l'effet devient réglable sans toucher au code.

### 1.4 L'affichage se fait sur `phy%`, donc à l'écran visible

```gfa
RC_COPY pic%,x&,y&,4,4 TO phy%,...
```

L'écran affiché est modifié pendant la construction, d'où un rendu qui « se
remplit » au lieu d'apparaître d'un coup. Construire dans `log%` puis un seul
`stab` par cycle donnerait une transition franche, et permettrait au passage de
supprimer les `stab` intermédiaires.

### 1.5 `maxcycle&=32` mais la boucle sort à 16

```gfa
EXIT IF (pixl_exit&=1) OR (cycle&>16)
```

La moitié des cycles n'est jamais exécutée. Soit c'est voulu et `maxcycle&` mérite
d'être mis à 16 pour que le code dise ce qu'il fait, soit c'est un reliquat de mise
au point.

### 1.6 La branche `box!` est cassée

```gfa
DEFFILL ! BYTE{pic_colors_array%+2*t160&(y&)+x&}
```

En GFA, `!` ouvre un commentaire : `DEFFILL` est donc appelé **sans argument**, et
`pic_colors_array%` renvoie au `DIM pic_colors|(64000)` commenté dans `hello`
(celui qui provoquait 4 bombes). Cette branche ne peut pas fonctionner en l'état —
à supprimer ou à réécrire.

### 1.7 Le blitter n'est pas exploité

`hello` détecte déjà la présence du blitter (STE/Falcon) et l'active. Sur les
machines qui en disposent, les recopies de blocs pourraient lui être confiées.
À évaluer seulement après les points 1.1 à 1.3 : le gain algorithmique est bien
plus important que le gain matériel.

---

## 2. Amélioration visuelle de la transition

Quelques idées, indépendantes de la performance :

- **Dézoom réel plutôt que pixelisation** : partir d'un bloc central agrandi et
  réduire la taille des blocs *en même temps* que la zone couverte s'étend.
  Coût identique si la réplication est faite par table + `BMOVE`.
- **Fondu de palette pendant la pixelisation** : `fadeon` existe déjà. Démarrer
  l'effet sur une palette sombre et la remonter au fil des cycles masquerait les
  paliers de la progression.
- **Entrelacer les rangées de blocs** (paires puis impaires) : donne une impression
  de résolution qui monte progressivement, sans coût supplémentaire.
- **Ne pas repasser par `taille&=1`** : le dernier cycle peut être remplacé par le
  `BMOVE pic%,phy%,32000` final, qui est déjà là et bien plus rapide.

---

## 3. Dette technique (rappel de `FEATURES.md`)

- Retirer l'instrumentation `chk$` et l'`ALERT` de `bye` une fois la version stable.
- Supprimer `pal.order.by.lum` (code mort, tri incorrect) ou le réécrire.
- Supprimer `TEST1.LST`.
- Réappliquer la restauration écran `XBIOS(5,L:oldlog%,…)` dans `bye` si besoin.

---

## 4. Question ouverte — mécanisme exact du bug `recolor_font`

Le correctif fonctionne (`ADNQ26T`), mais **la cause n'est pas prouvée**.

L'hypothèse retenue était l'accès au tableau global `t160&()` depuis la FUNCTION
`cptst` (`t160&` est `DIM`ensionné dans `pmul160`, pas dans `hello`). Ce qui a
réellement changé, c'est la **suppression de l'appel de FUNCTION dans la boucle** —
l'implication de `t160&()` reste une supposition.

Une vérification possible, si le sujet revient : garder l'appel à `cptst` mais lui
passer l'offset de ligne en paramètre au lieu de le faire lire `t160&()`. Si le
défaut réapparaît, le tableau est hors de cause et c'est l'appel de FUNCTION
lui-même qui pose problème.

Sans enjeu tant que la version actuelle tient — mais à ne pas oublier avant
d'utiliser `cptst`/`cpset` ailleurs dans le programme.

---

## 5. Fondus — réglages restants

### 5.1 Courbe gamma (si la rampe paraît encore inégale)

La rampe est linéaire en valeur de palette. La perception étant à peu près
logarithmique, une courbe légère peut aider. Table prête à coller, à lire une
fois au démarrage dans `hello` :

```gfa
DIM fade.step&(24)
RESTORE fade.ramp
FOR i&=0 TO 24
  READ fade.step&(i&)
NEXT i&
fade.ramp:
DATA 0,3,5,7,8,10,11,12,13,14,15,16,17,18,19,20,21,21,22,23,23,24,24,24,24
```

Puis dans les fondus, remplacer `i&` par `fade.step&(i&)` dans les trois lignes
de calcul. Mettre `DATA 0,1,2,3,…,24` redonne exactement le comportement actuel :
le réglage se fait donc sans toucher au code.

### 5.2 Durée

25 paliers à un `VSYNC` = 0,5 s. Pour un fondu plus lent, doubler le `VSYNC`
plutôt que le nombre de paliers : la palette n'a que 16 niveaux par composante,
au-delà les paliers supplémentaires ne changent rien visuellement.

### 5.3 `fadeoff` inclut le registre 0, `fadeon` non

C'est volontaire et asymétrique : à l'extinction on veut que **tout** s'éteigne,
y compris le fond ; à l'allumage le registre 0 est monté séparément par
`@fadereg0`, une fois l'image dessinée. Ne pas « corriger » cette asymétrie sans
relire §5.4.

### 5.4 Le registre 0 pendant `screen.opening`

`screen.opening` fait `~XBIOS(6,L:pal%)` — la palette entière d'un coup, sans
fondu. C'est le chemin de l'intro, où cela ne gêne pas. Si l'effet devait être
réutilisé sur les morceaux, il faudrait le passer en fondu comme le reste.

---

## 6. Autres pistes d'effets et transitions

### 6.1 Transitions alternatives, à coût comparable

- **Volet horizontal ou vertical** : `BMOVE` de lignes depuis `pic%` vers `log%`,
  une ou deux par trame. Quasi gratuit, très lisible à l'écran.
- **Fondu entrelacé** : afficher d'abord les lignes paires, puis les impaires.
  Deux `BMOVE` par trame, effet de « montée en résolution ».
- **Ouverture en bandes** depuis le centre : `screen.opening` fait déjà quelque
  chose de proche, le code est réutilisable.
- **Fondu croisé entre deux images** : impossible en palette indexée sans
  recalculer les pixels — à écarter.

### 6.2 Le rideau du texte — ✅ FAIT (`ADNQ26Z`)

`display.bitmap.txt` fait 8 `RC_COPY` par caractère (les 8 états du masque) plus
un `stab` par état, soit **16 changements d'écran par caractère**. Pour une ligne
de 30 caractères, cela fait 480 trames — c'est ce qui rend l'affichage du texte
lent. Deux pistes :
- dessiner **tous** les caractères à l'état *n* avant de passer à l'état *n+1*
  (le rideau devient global au lieu d'être lettre par lettre) : 8 trames au lieu
  de 480, et l'effet est plus élégant ;
- ou garder l'effet lettre par lettre mais sans `stab` intermédiaire.

### 6.3 Optimisations générales

- **`t160&()` est un tableau de mots** : `CARD{t160&(y&)}` force une extension de
  signe. Un tableau de longs (`t160%()`) éviterait la conversion à chaque accès.
- **`DIV` et `MUL` dans les fondus** : ~1 ms par palier, négligeable. Ne pas
  optimiser ici, le gain serait invisible.
- **`RC_COPY` plein écran pour effacer** (`RC_COPY log%,0,0,320,200 TO log%,0,0,0`)
  : un `BMOVE` depuis un buffer noir, ou un remplissage direct, serait plus rapide.
- Le vrai poste reste **§1.1**, la pixelisation.


---

## 7. Suite — alternance des effets de révélation

`@scrambling` était appelé **depuis** `pixelisation` ; il est désormais appelé
depuis `quizz`, à côté de `@pixelisation`. Les effets sont donc réordonnables
sans toucher aux procédures.

Pour vraiment alterner d'un morceau à l'autre, il reste à **donner à chaque effet
la même interface de durée** que `pixelisation(duree&)` :

### Audit vérifié (2026-08-02, sur `ADNQ27B.LST`)

| Effet | État réel | Ce qui manque |
|---|---|---|
| `pixelisation` | ✅ complet | pilotée par `duree&`, 9 niveaux × 20 bandes |
| `scrambling` | ✅ complet (`ADNQ27C`) | interface `(duree&)`, 100 cycles, sortie à l'espace |
| `random_pixels` | ✅ réécrite (`ADNQ27F`) | blocs 16×4, Fisher-Yates sur 1000 mots |
| `slow_fadeon` | ❌ **coquille vide** | corps = `' to implement`, 2 lignes |
| `dysplay_text_shuffle` | ⚠️ hors sujet | signature `(background%,text%)` — effet de **texte**, pas de révélation d'image |
| `dezoom.rc` (ancien `RC_COPY`) | ✅ restauré + corrigé (`ADNQ27G`) | 9 paliers utiles au lieu de 6 dont 2 morts |
| Volet / entrelacement | ⬜ non écrits | cf. §6.1 |

#### `random_pixels` — réécrite (`ADNQ27F`)

La version d'origine cumulait quatre défauts, dont **trois vrais bugs** :

```gfa
y%=RANDOM(200)
pos%=t160&(y&)+t160&(y%)+x&           ← y& jamais affecté ici, et compté en double
@cpset(x&,y&,phy%,cptst(x&,y&,pic%))  ← utilise y& (périmé), pas y%
```

Plus un **échantillonnage par rejet** (`REPEAT … UNTIL done!(pos%)=0`) qui devient
pathologique en fin de parcours — les derniers blocs demandent un nombre énorme de
tirages — et le `DIM done!(64000)` documenté ci-dessous.

Réécriture : l'unité dévoilée est un bloc de **16×4 pixels** (un mot par plan sur
quatre lignes), soit 20 × 50 = **1000 blocs**. L'ordre vient d'un Fisher-Yates sur
1000 mots (2 Ko). Chaque bloc est visité une fois et une seule, **sans aucun rejet**.

Vérifié : les 32000 octets de l'écran sont écrits **exactement une fois** chacun,
offset maximal 32000, et `MUL(by&,640)` plafonne à 31360, sous la limite du mot
signé.

#### Motif du `DIM` à 64000 — pour mémoire

```gfa
DIM done!(64000)        ! l. 1303
```

C'est **exactement le motif** que `hello` documente comme provoquant quatre
bombes :

```gfa
' v 4 bombs executing this v why ?!! 64000<65536 !
' DIM pic_colors|(64000) ! ... 4 bombes ... plantage des lancement prog ??!!!
```

La procédure `DIM` par ailleurs **deux fois** le même tableau (l. 1303 puis 1310
avec `maxpixels%-1`). Elle attend aussi une durée en **secondes** (`duree%=120`)
alors que `pixelisation` raisonne en trames, et son `maxcycle&=100` est figé.

À réécrire sans tableau de 64000 entrées avant tout essai — un générateur
pseudo-aléatoire parcourant les pixels sans mémoire d'état ferait l'affaire.

#### L'ancien dézoom `RC_COPY` a bien disparu

Il a été **remplacé**, pas conservé à côté. Il reste récupérable :

- `ADNQ26Y.LST` (et versions antérieures) contiennent la version d'origine ;
- `git show 57a5c8f^:ADNQ26Z.LST` donne le fichier juste avant le remplacement.

Pour l'utiliser en alternance, le remettre sous un **nom distinct**
(`pixelisation.rccopy` par exemple) plutôt que de le restaurer à sa place.

Réserve : sa lenteur venait des ~4500 `RC_COPY` par niveau, et sa durée était
**subie**. Comme effet alternatif il faudra lui donner la même interface
`(duree&)` que les autres, sinon il ne tiendra pas les 2 minutes de façon réglée.

#### Fin progressive commune — `@dissolve.to.pic(trames&)`

Appuyer sur espace ne révèle **jamais** l'image d'un coup : l'effet en cours est
abandonné et l'image se dévoile en **dissolve**, ligne par ligne dans un ordre
aléatoire, sur 100 trames = **2,00 s** (vérifié : 200 lignes distinctes,
couverture complète).

C'est particulièrement utile pour les effets **naturellement lents** — le dézoom
`RC_COPY` du chantier 2 notamment. On ne cherche pas à les accélérer, ce qui est
impossible vu leur coût machine : on bascule sur cette révélation-ci, progressive
et de durée connue.

`trames&` règle la vitesse : 50 → 1 s (4 lignes/trame), 100 → 2 s (2 lignes),
150 → 4 s (1 ligne, plancher).

#### Interface commune à viser

```gfa
PROCEDURE <effet>(duree&)   ! duree& en trames, 50 par seconde
  ' doit consommer duree& trames, et sortir tot si pixl_exit&=1
```

Une fois les quatre effets alignés sur cette signature, le sélecteur dans `quizz`
tient en quelques lignes :

```gfa
SELECT MOD(file&,4)
CASE 0
  @pixelisation(pix.duree&)
CASE 1
  @scrambling(pix.duree&)
CASE 2
  @dezoom.rc(600)
CASE 3
  @random_pixels(pix.duree&)      ! chantier 3, pas encore fait
ENDSELECT
```

**État : `SELECT MOD(file&,3)` est en place** avec les trois premiers effets.
Passer à `MOD(file&,4)` quand `random_pixels` sera réécrite.

Une fois toutes les procédures alignées, un simple sélecteur dans `quizz` suffit :

```gfa
IF EVEN(file&)
  @pixelisation(pix.duree&)
ELSE
  @scrambling(pix.duree&)
ENDIF
```

### Réglage de la durée et de la granularité

`pix.duree&` est défini dans `hello` : **6000 trames = 120 s** à 50 Hz.

**9 niveaux × 8 bandes = 72 étapes visibles**, soit une modification à l'écran
toutes les **1,7 s**. La première version n'avait que 5 niveaux tenus 24 s : rien
ne bougeait pendant très longtemps.

Ce qui a permis de passer de 5 à 9 niveaux : la taille **verticale** des blocs
n'est pas contrainte aux puissances de deux — c'est seulement le nombre de lignes
répliquées par `BMOVE`. Seule la taille **horizontale** dépend des tables. D'où
des paliers rectangulaires intercalés :

```
16x16 -> 16x8 -> 8x8 -> 8x4 -> 4x4 -> 4x2 -> 2x2 -> 2x1 -> 1x1
```

**Porté à 20 bandes de 10 lignes** dans `ADNQ27B` : 9 × 20 = **180 étapes**, une
modification toutes les **0,67 s**. Les diviseurs entiers de 200 utilisables sont
8, 10, 20 et 25 bandes — 16 bandes donneraient 12,5 lignes, à éviter.

### Optimisations de `pixdraw` appliquées (`ADNQ27B`)

| Optimisation | Gain |
|---|---|
| Transformation horizontale au **mot** (masque + décalages) au lieu des tables d'octets | 80 itérations par ligne au lieu de 160 |
| Réplication verticale par **doublement** (1, 2, 4, 8 lignes) | 4 `BMOVE` au lieu de 15 pour `nv&=16` |
| Niveau `1×1` en un seul `BMOVE` de 32000 | 1 appel au lieu de 200 |

Le principe de la transformation au mot : le masque isole le bit de gauche de
chaque bloc, les `OR` décalés le recopient sur les bits à sa droite.

```gfa
w%=CARD{sl%+i&} AND &HAAAA     ! blocs de 2 : bits 15,13,11,...
CARD{dl%+i&}=w% OR SHR(w%,1)
```

Masques : `&HAAAA` pour 2 pixels, `&H8888` pour 4, `&H8080` pour 8. Équivalence
avec les anciennes tables vérifiée sur 20000 mots aléatoires, 0 divergence.

`w%` est un **long** et non un mot : `&HAAAA` a le bit 15 à 1, un `SHR&` sur un
mot signé propagerait ce bit. Ne pas « optimiser » ce type.

La table des niveaux est dans `@pixinit` (`DATA 16,16,16,8,…`) : ajouter des
paliers ne demande que d'allonger cette ligne et d'ajuster les bornes.

### Feu d'artifice

L'image finale `FIREWORK.PI1` est aujourd'hui affichée telle quelle. L'idée d'en
faire un effet de démo plutôt qu'une image fixe est notée pour la prochaine
version — les briques existent déjà : `@pixdraw` pour les blocs, `@fadereg0` et
les fondus pour la palette, `@scrambling` pour les lignes.


---

## 8. Effets sur le texte artiste / titre

Idées classées par rapport impact / coût. Les deux premières se greffent sur le
code existant sans rien restructurer.

### 8.1 Ondulation verticale des lettres — le meilleur rapport

`display.bitmap.txt` dessine déjà chaque caractère à une position calculée. Il
suffit de décaler son `y` par une table de sinus indexée par (rang du caractère
+ phase). Coût quasi nul : les `RC_COPY` sont déjà là, seule l'ordonnée change.

```gfa
' dans hello, une fois
DIM sinus&(15)
RESTORE sin.vague
FOR i&=0 TO 15
  READ sinus&(i&)
NEXT i&
sin.vague:
DATA 0,1,2,3,3,3,2,1,0,-1,-2,-3,-3,-3,-2,-1

' dans display.bitmap.txt, remplacer yb& par :
yv&=yb&+sinus&(AND(ADD(char&,phase&),15))
```

En incrémentant `phase&` à chaque trame, la vague se déplace le long du texte.
Attention : `area.lines.erase` doit effacer une bande **plus haute** (± 3 lignes)
pour que les lettres qui montent ne laissent pas de traînée.

### 8.2 Cyclage de palette sur l'encre — le moins cher

Le texte est dessiné dans `ink_col&`. Faire varier ce seul registre matériel
d'une trame à l'autre fait pulser le texte, pour **une écriture de palette par
trame** :

```gfa
CARD{&HFFFF8240+ink_col&+ink_col&}=grad&(AND(phase&,15))
```

`grad&()` : une rampe de 16 couleurs préparée au chargement de l'image (par
exemple de la couleur la plus claire vers la plus contrastée). Se combine
parfaitement avec 8.1.

### 8.3 Défilement horizontal matériel (STE)

Votre machine est une **STE** : le registre `$FF8265` (0-15) décale l'affichage
horizontalement par pas d'un pixel, en matériel. Associé à une interruption
Timer B ligne par ligne, il donne un scroller ou une distorsion sinusoïdale
**gratuits** côté processeur.

Réserve : demande une maîtrise du timing raster, et **casse la compatibilité
STF**. À réserver à une version « démo » assumée.

### 8.4 Barres raster derrière le texte

Changer le registre 0 à chaque ligne balayée sur les 16 lignes du texte donne les
barres de couleur caractéristiques de la scène ST. Même réserve que 8.3 : Timer B
et timing précis.

### 8.5 Lettres qui tombent

Chaque caractère part au-dessus de sa position et descend jusqu'à sa ligne, avec
un décalage de départ par lettre. Réutilise le rideau existant, il suffit de
faire varier `y` au lieu de l'état du masque.

### 8.6 Étirement / zoom du bandeau

Appliquer la réplication de `@pixdraw` à la seule bande de texte : le titre
apparaît étiré puis se resserre. Les briques existent, il faut juste borner la
zone traitée.

---

**Recommandation** : commencer par **8.1 + 8.2 combinés**. C'est peu de code, ça
ne touche ni au timing raster ni à la compatibilité STF, et l'effet obtenu est
déjà très « démo ». 8.3 et 8.4 méritent leur propre cycle, avec un repli prévu
pour les machines STF.


---

## 9. Points tranchés (2026-08-03)

### 9.1 Fondu après l'écran d'intro — il existe

Vérifié dans `quizz` :

```
REPEAT ... UNTIL ENTREE     ! attente sur l'ecran d'intro
@ldpic(...)                 ! charge image + palette dans pal% UNIQUEMENT
@set_text_colors
@fadeoff                    ! <- le fondu est bien la
@clearpal
```

`ldpic` n'écrit **jamais** dans la palette matérielle (`BMOVE adr%+2,pal%,32`),
donc aucun saut de couleur ne masque le fondu.

Ce qui a changé, c'est sa **perception** : l'ancien fondu séquentiel
rouge → magenta → vert était très voyant. Le nouveau est une rampe de luminosité
propre sur 25 trames, soit **0,5 s** — correct mais discret.

Correctif si on le veut plus présent : ajouter un second `VSYNC` par palier
(1 s au lieu de 0,5 s), ou faire du nombre de paliers un paramètre comme pour les
effets de révélation.

### 9.2 Fondu du volume musical — pas faisable proprement

Même conclusion que pour le raster (§8.3) : **il faut de l'assembleur**.

Le lecteur SNDH pilote directement le YM2149 et **réécrit les registres de volume
(8, 9, 10 via `$FF8800`) à chaque trame, sous interruption**. Toute écriture de
volume depuis GFA serait écrasée dans la trame suivante.

Trois voies, toutes hors GFA pur :
- patcher le lecteur pour qu'il applique un facteur d'atténuation — assembleur ;
- utiliser un lecteur SNDH exposant un point d'entrée « volume » — non standard ;
- le registre de volume maître STE `$FF8924` ne concerne que le **son DMA**, pas
  le YM : sans effet sur du SNDH.

Repli sans coût : couper net avec `@sndh.off` en même temps que le fondu image,
la coupure passe inaperçue si l'écran s'éteint simultanément.

### 9.3 Ressources notées

- `2p_16_22.pi1` (32 066 o) — police prévue pour le scrolltext de l'intro.
- `GFX_WIP/` et `OLD/` sont désormais dans `.gitignore`.

### 9.4 Prochains chantiers convenus

1. **Ondulation animée du texte** jusqu'à la réaction clavier, avec
   **restauration ciblée du fond** (garder une copie de la bande d'image dans
   `lowborderm%`, la restaurer avant chaque redessin). Vise 25 Hz — redessiner
   deux lignes de texte coûte environ 80 `RC_COPY`, ce qui ne tient pas en une
   trame mais passe en deux.
2. **Feu d'artifice** en remplacement de l'image finale digitalisée.


---

## 10. Audit du 2026-08-03 — défauts restants

Rien de bloquant, mais voici ce qui ressort d'un balayage du fichier.

### 10.1 Variables de boucle globales — le risque le plus sérieux

**44 procédures, seulement 19 déclarent des `LOCAL`.** Les autres utilisent des
variables globales comme compteurs (`i&`, `y&`, `x&`, `char&`…). Tant qu'aucune
procédure n'en appelle une autre qui réutilise le même nom, cela fonctionne — mais
c'est exactement le genre de piège qui se déclenche à la première réorganisation.

`display.bitmap.txt` appelle `area.lines.erase` : la première utilise `char&`,
`m.appear&`, `cx&`, `cy&`, `yv&` ; la seconde `x1&`, `y1&`, `x2&`, `y2&`, `w&`,
`h&`, `y&`, `bl&`. Pas de collision **aujourd'hui**, mais rien ne l'empêche.

À traiter procédure par procédure, en commençant par celles qui s'appellent entre
elles. Sans urgence, mais à ne pas oublier avant d'ajouter des effets imbriqués.

### 10.2 Instrumentation de debug toujours en place

- **15 affectations `chk$=`** dans le flux principal et `set_text_colors`.
- L'`ALERT` de la ligne 355 s'affiche **même en sortie normale** : elle est dans
  le gestionnaire `ON ERROR GOSUB bye`, mais `bye` est aussi le chemin de sortie
  volontaire.

À retirer une fois la version stabilisée. `chk$` reste utile tant qu'on débogue.

### 10.3 Code mort confirmé

| Procédure | Références hors définition |
|---|---|
| `slow_fadeon` | **0** — coquille vide (`' to implement`) |
| `pal.order.by.lum` | **0** — tri incorrect de surcroît, cf. §3 |
| `dysplay_text_shuffle` | 1 (définition seule) |
| `create_outline_font` | 1 (définition seule) |
| `fpset` | 1 (définition seule) |

`create_outline_font` mérite peut-être d'être conservée : un contour logiciel
autour des lettres est une alternative sérieuse au bandeau noir (cf. §10.4).

### 10.4 Piste : contour plutôt que bandeau

Plutôt que d'effacer un gros bloc derrière le texte, `create_outline_font`
pourrait dessiner un contour d'une couleur contrastée autour de chaque glyphe. Le
texte deviendrait lisible **sur l'image elle-même**, sans bandeau. C'est la voie
vers l'affichage « en transparence » évoqué.

Réserve : le contour consomme un second index de couleur, et il faut vérifier
qu'il contraste à la fois avec l'encre et avec le fond local.

### 10.5 Points vérifiés, rien à signaler

- **Syntaxe `RC_COPY`** : `RC_COPY s_adr,sx,sy,w,h TO d_adr,dx,dy[,m]` — conforme
  au manuel, l'usage dans le code est correct partout. Sur STe le blitter accélère
  effectivement ces copies, et `hello` l'active déjà quand il le détecte.
- **Aucune division flottante** ne subsiste dans une boucle chaude.
- `FOR`/`NEXT`, `IF`/`ENDIF`, `WHILE`/`WEND`, `REPEAT`/`UNTIL` : tous équilibrés.


---

## 11bis. Optimisations mesurées, non encore appliquées

### `pixdraw` en LONG — facteur 2 supplémentaire

La transformation horizontale traite un **mot** à la fois. En **long**, elle en
traite deux : **40 itérations par ligne au lieu de 80**.

```gfa
w%=LONG{sl%+i&} AND &HAAAAAAAA     ! blocs de 2
LONG{dl%+i&}=w% OR SHR(w%,1)
```

Masques : `&HAAAAAAAA` (2 px), `&H88888888` (4), `&H80808080` (8), `&H80008000` (16).

**Le point délicat, vérifié** : deux mots consécutifs sont deux **plans
différents** du même groupe de 16 pixels. Un décalage pourrait faire fuir un bit
d'un plan vers l'autre. Ce n'est pas le cas — les masques ne conservent que des
bits assez éloignés de la frontière pour que l'étalement reste dans son mot.
Testé sur **30 000 longs aléatoires, 0 divergence** pour les quatre tailles.

Attention : les adresses écran sont paires, donc l'accès `LONG` est légal.

### Ne redessiner que les caractères qui bougent — gain réel : 25 %

J'avais avancé 75 %, c'était faux. Avec la table
`0,1,2,3,3,3,2,1,0,-1,-2,-3,-3,-3,-2,-1`, **12 caractères sur 16** changent
d'ordonnée entre deux phases. Le gain ne vaut probablement pas la complexité.

### Bandeau noir — presque inutile

Le masque `REPLACE` force déjà chaque cellule 8×8 à la couleur 0 avant le `AND`.
Le bandeau ne couvre plus que la marge de 2 pixels autour du texte. Passer
`bg_erase!` à `FALSE` gagnerait ce liseré.

Pour une **vraie** transparence il faudrait abandonner le couple masque + `AND`
au profit d'un mode `OR` (7), ce qui change la nature de l'effet : en palette
indexée, `fond OR encre` donne des couleurs imprévisibles. À étudier séparément.

---

## 11. Ondulation animée avec restauration ciblée du fond

**Le principe qui rend tout simple** : le fond sous le texte, c'est l'image
elle-même, déjà en mémoire dans `pic%`. Pas besoin de tampon de sauvegarde —
restaurer une ligne, c'est `BMOVE pic%+t160&(y),phy%+t160&(y),160`.

Boucle d'animation, à la place de l'attente clavier actuelle dans `quizz` :

```
1. restaurer la bande depuis pic%        (~40 BMOVE de 160 octets)
2. dessiner les deux lignes a la phase courante  (~80 RC_COPY)
3. INC phase&
4. VSYNC, VSYNC                          (25 Hz)
5. tester le clavier, sortir si touche
```

**Budget** : environ 80 `RC_COPY` par trame ne tient pas en 20 ms, mais passe en
40 ms — d'où les deux `VSYNC`. À mesurer sur machine.

**Effet de bord bienvenu** : en restaurant depuis `pic%` plutôt qu'en peignant un
bandeau, le texte se retrouve **sur l'image**, ce qui est exactement l'affichage
en transparence visé. Le bandeau `box_col&` devient optionnel.

**Point d'attention** : la bande restaurée doit couvrir l'amplitude de la vague
(± 3 lignes) plus la hauteur des glyphes, sinon traînée.

---

## 12. Feu d'artifice

### 12.1 Principe retenu

**Particules précalculées.** Les trajectoires sont calculées une seule fois au
démarrage dans un tableau (x, y par trame) ; à l'exécution il ne reste que
l'affichage et l'effacement. Une gerbe = N points avec vitesse initiale et
gravité.

### 12.2 Ce qui rend l'effet beau pour presque rien

- **Cyclage de palette pour le refroidissement** : blanc → jaune → orange → rouge
  → noir. Les particules ne sont jamais redessinées, seule la palette change —
  une écriture par trame. C'est ici que le cyclage est légitime, contrairement au
  texte (cf. §13) : les particules sont les seules à utiliser ces registres.
- **Effacement sélectif** : restaurer uniquement les pixels touchés depuis le fond,
  jamais un `CLS` global.
- **Gerbes décalées dans le temps** : trois ou quatre départs échelonnés donnent
  une impression de densité sans multiplier le coût instantané.

### 12.3 Écran d'intro — projets notés

L'image d'intro sera remplacée par du pixel art personnel. Effets envisagés :
scrolltext en bas d'écran (police `2p_16_22.pi1`, 32 066 o), 3 VU-mètres temps
réel, note de musique en 3D fil de fer.

---

## 13. Encre choisie par rareté × contraste

Le cyclage de palette sur l'encre du texte **ne marche pas** tel quel : changer un
registre repeint tous les pixels de l'image qui l'utilisent.

**Mais la mesure ouvre une voie** : l'index le **moins utilisé** de vos images ne
couvre en moyenne que **1,39 %** des pixels.

```
01.PI1 : index 15 →  0,42 %
06.PI1 : index 12 →  0,85 %
02.PI1 : index 14 →  1,11 %
```

Si `test_color_contrast` combinait **rareté et contraste** au lieu du seul
contraste, animer ce registre ne toucherait qu'environ 1 % de l'image — invisible
en pratique. Le code contient déjà `DIM freq%(16) ! for scan_pic_for_color` :
l'infrastructure était prévue.

Compromis à arbitrer : l'index le plus rare n'est pas forcément le mieux contrasté.
Un score pondéré, à régler à l'œil.
