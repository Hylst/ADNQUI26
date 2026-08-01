# TODO — pistes d'amélioration

## 1. Effet de transition (`pixelisation`, l. 403)

L'effet fonctionne mais il est très lent. Voici ce que le code fait aujourd'hui,
et ce qui coûte cher — par ordre de gain décroissant.

### 1.1 Le coût réel : un `RC_COPY` par bloc de 4×4

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
