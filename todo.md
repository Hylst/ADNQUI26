# TODO — pistes d'amélioration

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
| `random_pixels` | ❌ **jamais appelée, et risquée** | voir ci-dessous |
| `slow_fadeon` | ❌ **coquille vide** | corps = `' to implement`, 2 lignes |
| `dysplay_text_shuffle` | ⚠️ hors sujet | signature `(background%,text%)` — effet de **texte**, pas de révélation d'image |
| Ancien dézoom `RC_COPY` | ❌ **supprimé** du fichier courant | récupérable, voir plus bas |
| Volet / entrelacement | ⬜ non écrits | cf. §6.1 |

#### `random_pixels` — à ne pas activer en l'état

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
  @pixelisation.rccopy(pix.duree&)
CASE 3
  @random_pixels(pix.duree&)
ENDSELECT
```

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
