# Fonctionnalités — ADN QUIZZ

Légende : ✅ fonctionne · ⚠️ fonctionne avec réserve · 🔬 modifié, non testé · ❌ en panne · ⬜ non implémenté

## Déroulé du quiz

| Fonctionnalité | État | Note |
|---|---|---|
| Enchaînement des 30 morceaux | ✅ | |
| Lecture musique SNDH | ⚠️ | « Pas de musique » signalé lors d'un test, jamais investigué. Vérifier d'abord la config audio de Steem SSE (YM/DMA, volume). **Piste sérieuse** : le début de `sndhplay%` (186 octets) peut être écrasé par la fin de `pic%` — correctif dans `ADNQ26R.LST`, cf. `STRUCT.md`. |
| Apparition progressive de l'image (pixellisation, scrambling) | 🔬 | Réécrite dans `ADNQ26Z` : dessin par plans de bits, durée pilotée (`pix.duree&` = 6000 trames = 120 s). |
| Affichage du texte réponse (artiste / titre) | ⚠️ | Lisible seulement si la palette de l'image s'y prête — c'est le sujet ouvert ci-dessous. |
| Contrôle clavier (Espace / Entrée / Backspace) | ✅ | |
| Fondus de palette (`fadeon` / `fadeoff`) | ✅ | Trois défauts corrigés dans `ADNQ26V`, voir ci-dessous. |
| Écran d'intro, écran final « fireworks » | ✅ | |

## Le sujet ouvert : contraste du texte

**Le problème.** `DATA\FONT.INL` est une fonte bitmap 8×8 à **deux couleurs**
seulement : l'index **0** (fond, 3580 pixels) et l'index **2** (encre, 1540 pixels)
— vérifié en décodant le fichier. Chaque `.PI1` embarque sa propre palette, donc
le registre 2 a un RVB différent à chaque image : parfois lisible, parfois
illisible (typiquement sombre sur sombre).

> ⚠️ `continue.md` §2 affirme « index 15 » et le commentaire du bloc `DATA`
> annonce « 4 colours » : **les deux sont faux**. Se fier au fichier.

**Contrainte posée par l'utilisateur** : ne pas modifier les images, et ne pas
sacrifier 2 des 16 couleurs — cela dégraderait le rendu.

**Ce sur quoi le mécanisme repose** (vérifié, cf. `AGENTS.md`) : le fond de chaque
cellule 8×8 est **toujours le registre matériel 0**, imposé par le AND de
`RC_COPY`. C'est donc avec le registre 0 que l'encre doit contraster.

### Chaîne de traitement

| Étape | État | Note |
|---|---|---|
| `test_color_contrast` lit la palette dans `pal%` | ✅ | Plus rapide et indépendant du timing du fondu (avant : `XBIOS(7,…)`). |
| Appel après chaque `ldpic` (intro, chaque morceau, finale) | ✅ | |
| `recolor_font` repeint l'encre | ✅ | Logique de `cptst`/`cpset` **recopiée dans la procédure** depuis `ADNQ26T` : plus d'appel de FUNCTION, plus de dépendance à `t160&()`. |
| Conversion VDI dans `area.lines.erase` | ✅ | `COLOR vdicol|(box_col&)` |
| Poids des plans dans `cptst` / `cpset` | 🔬 | **Corrigé le 2026-08-01, non testé.** Voir ci-dessous. |
| Choix `test_color_contrast(2,…)` au lieu de `(0,…)` | 🔬 | **Non testé.** Seul écart de code avec la version qui tourne, avant la correction des plans. |

### Correction des plans de bits (2026-08-01) — à tester

`cptst` et `cpset` attribuaient le poids **8** au 1er plan et le poids **1** au 4ᵉ,
soit l'inverse de la réalité matérielle. Les deux fonctions étant cohérentes entre
elles, la recopie pixel-à-pixel fonctionnait ; mais `recolor_font(ink_col&)` leur
passe un **vrai numéro de registre**, qui se retrouvait donc peint **bit-inversé** :

| `ink_col&` calculé | couleur réellement peinte |
|---|---|
| 1 | 8 |
| 2 | 4 |
| 4 | 2 |
| 0, 6, 9, 15 | inchangées (palindromes binaires) |

L'encre d'origine étant la couleur **2**, elle était repeinte en **4** au lieu de
la couleur demandée : le correctif de contraste ne pouvait donc pas fonctionner.

Tant que ce défaut était présent, tester `choix 2` ne pouvait rien démontrer —
la couleur choisie était brouillée avant d'être appliquée.

### Palette STE mal lue (2026-08-01) — corrigé dans `ADNQ26S`

Les `.PI1` de ce projet sont au **format STE** (4096 couleurs) : les bits 11, 7
et 3 y sont utilisés — `$0EC5`, `$0F77`… impossibles en format ST 3 bits.

Or `test_color_contrast` lisait les quartets bruts. En STE, le 4ᵉ bit de chaque
composante est le **LSB stocké en tête du quartet** :

```
$0EC5  lu RGB(14,12,5)  ->  vraie couleur RGB(13,9,10)
$0180  lu RGB(1,8,0)    ->  vraie couleur RGB(2,1,0)
```

Le classement par luminosité était donc calculé sur des valeurs brouillées.
Correction : `valeur = SHL&((n& AND 7),1) + SHR&((n& AND 8),3)`. Sur une palette
ST pure (quartets 0-7) elle rend `2*n`, ce qui laisse le classement inchangé —
elle est donc sans risque.

Effet mesuré sur les 32 images : **25 changent de couleur**. Les résultats
deviennent cohérents — `ink_col&` = **15** (le plus clair) et `box_col&` = **0**
(le plus sombre) pour les 30 morceaux, au lieu de valeurs dispersées
(12, 6, 14, 11, 1, 5, 3…) qui n'étaient que du bruit.

### Fragmentation des glyphes — CAUSE IDENTIFIÉE (corrigée dans `ADNQ26T`)

Symptôme : texte net sur l'écran d'intro, lettres trouées dès le 1ᵉʳ morceau.

**Cause — hypothèse, non prouvée.** `recolor_font` appelait `cptst`, une
**FUNCTION** qui lit le tableau global `t160&()`. Or `t160&(200)` est dimensionné **dans `pmul160`**, pas dans les
`DIM` groupés de `hello` — le commentaire du code s'en méfiait déjà (« bugs with
DIM in functions or procedures ?? »). C'est la cause d'`ERR 15` déjà documentée
dans `continue.md` §6.2. Quand `t160&(y&)` rend une valeur erronée, `cptst` lit
**et `cpset` écrit** à la mauvaise adresse : la conversion est partielle.

**Pourquoi l'intro était épargnée** — le point qui a permis de trancher :

| Écran | `ink_col&` | Effet de `recolor_font` |
|---|---|---|
| Intro (`adnquizz.pi1`) | **2** | = la couleur d'origine de `FONT.INL` → **no-op**, le défaut reste invisible |
| Les 30 morceaux | **15** | doit allumer **3 plans vides** sur 1540 pixels → le défaut devient visible |

**Correction** (`ADNQ26T`) : la logique de `cptst`/`cpset` est recopiée dans
`recolor_font`, qui calcule l'offset de ligne directement (`font%+MUL(y&,160)`).
Plus aucun appel de FUNCTION, plus aucune dépendance à un tableau global.
C'est le contournement que `continue.md` §6.2 avait validé et qui avait été perdu
au retour à la base P.

Équivalence vérifiée par simulation : l'ancienne et la nouvelle logique donnent
le même résultat exact (3580 pixels de fond, 1540 d'encre en couleur 15).
**Testé sur machine : le texte est net.** ✅

> ⚠️ Ce qui a réellement changé, c'est la **suppression de l'appel de FUNCTION
> dans la boucle**. L'implication de `t160&()` reste une supposition — voir
> `todo.md` §4 pour la vérification qui trancherait.

**Quatre hypothèses ont été vérifiées et écartées en chemin** — utile pour ne pas
les reprendre :

| Hypothèse | Verdict |
|---|---|
| Le masque du rideau serait incomplet | **Faux.** Simulé : les états 6 et 7 couvrent 64/64 pixels. |
| La correction des poids de plans | **Impossible.** Elle ne change qu'une couleur uniforme. |
| `stab` alterne les tampons, l'un serait en retard | **Faux.** Les deux états finaux sont complets. |
| `choix 2` au lieu de `choix 0` | **Hors de cause** : les deux donnent 12 sur l'image 01. |

### Fondus de palette — trois défauts corrigés (`ADNQ26V`)

**1. Le fondu ne retombait jamais sur la palette de l'image.** `fadeon` et
`fadeoff` décodaient avec `AND 7`, ce qui jette le 4ᵉ bit des composantes STE.
Vérifié sur l'image 01 :

```
couleur 12 : image $0EC5  ->  l'ancien fondu finissait sur $0645
couleur 15 : image $0F77  ->  l'ancien fondu finissait sur $0777
```

À la fin de chaque montée, la palette réelle était ensuite rétablie ailleurs
(`~XBIOS(6,L:pal%)`), d'où un **saut de couleur systématique**. Le décodage STE
corrige cela : la dernière étape tombe désormais **exactement** sur la palette de
l'image (vérifié sur les 16 couleurs), et la rampe passe de 8 à **16 niveaux**
(0, 2, 4, 6, 9, 11, 13, 15 — monotone).

**2. Le flash blanc — vraie cause (`ADNQ26X`).** Le fondu n'était pas une rampe
de luminosité mais un **balayage de canaux** : la boucle `ty&` montait le rouge
seul, puis le bleu, puis le vert.

| Phase | Ce qui monte | Écran |
|---|---|---|
| `ty&=0` | rouge seul | noir → **rouge** |
| `ty&=1` | bleu | rouge → **magenta** |
| `ty&=2` | vert | magenta → couleur complète |

Le vert (poids perceptuel 0,59) arrivant en dernier sur un rouge et un bleu déjà
pleins produisait une montée brutale vers le blanc. Luminosité moyenne des 16
couleurs, trame par trame :

```
ANCIEN  : 0 0 1 1 2 2 2 3 3 3 3 3 3 3 3 3 3 4 4 5 5 6 6 7
                          plateau de 9 trames    puis surge
NOUVEAU : 0 0 0 0 1 1 1 2 2 2 2 3 3 3 4 4 4 4 5 5 5 6 6 6 7
```

Le palier correspond à la phase bleue, dont le poids n'est que 0,11 : l'œil ne
voit presque rien bouger pendant 9 trames, puis tout arrive d'un coup.

**Correction** : les trois composantes évoluent **ensemble**, 25 pas (0 à 24) pour
conserver la durée d'origine. Rampe monotone, arrivée exacte sur la palette de
l'image. Plus aucun flottant : `DIV(MUL(r&(ii&),i&),24)` — ~1 ms par pas dans une
trame de 20 ms.

> Une première tentative (bornage `@clampcol` contre un `cr&` négatif issu de
> l'arithmétique flottante) **n'a pas suffi** : l'hypothèse était plausible mais
> ce n'était pas la cause. Le bornage est conservé, il ne coûte rien et protège
> l'encodage.

**3. `cv&` et `cb&` n'étaient pas initialisés** dans les branches `ty&=0` et
`ty&=1` de `fadeon`. Ce sont des variables **globales** : elles gardaient la
valeur laissée par le fondu précédent. En pratique `fadeoff` les laissait à 0,
donc le défaut était masqué — mais le résultat dépendait de l'appel antérieur.
Les trois composantes sont maintenant affectées dans chaque branche.

Le nouvel encodage reste correct sur **STF** : le matériel y ignore le bit 3 de
chaque quartet, donc il lit les 3 bits de poids fort, soit la bonne valeur.

### Flash résiduel — le registre 0 (`ADNQ26Y`)

`fadeon` est appelé juste après `RC_COPY log%,… TO log%,0,0,0` : l'écran est alors
**entièrement en couleur 0**, et `pixelisation` ne dessine l'image qu'ensuite.
Faire monter le registre 0 revient donc à laver tout l'écran de la couleur de fond
de l'image. Mesuré sur les 33 images :

| Image | Registre 0 | Luminosité |
|---|---|---|
| `10.PI1` | RGB(7,3,2) | 4,09 — flash net |
| `29.PI1`, `08.PI1` | — | 3,9 / 3,2 |
| `03`, `07`, `20.PI1` | RGB(0,0,0) | 0,00 — aucun flash |

D'où le « seulement certaines images ». `fadeon` ramène désormais les registres
**1 à 15** seulement ; le registre 0 est monté par `@fadereg0` **après**
`pixelisation`, quand l'image occupe déjà l'écran — seules les zones de fond
s'éclaircissent alors.

### Rampe : arrondi au lieu de troncature (`ADNQ26Y`)

`DIV(MUL(r&,i&),24)` tronquait : une composante de valeur 2 restait à 0 jusqu'au
pas 12, puis rattrapait d'un coup. D'où un début qui paraissait lent et une fin
rapide. Avec `DIV(ADD(MUL(r&,i&),12),24)` :

| Mesure | Troncature | Arrondi |
|---|---|---|
| 1ᵉʳ mouvement d'une composante faible | pas 12 | **pas 6** |
| Régularité des paliers (écart-type) | 0,203 | **0,100** |
| Dernier saut de la rampe | 1,0 | **0,1** |

### Version courante : `ADNQ27G.LST`

Le développement se fait désormais **sur ce seul fichier**, avec un commit git à
chaque étape pour permettre un retour arrière. Plus de nouvelle lettre par
modification.

### File d'attente des cycles de test

| Version | Contenu | À vérifier |
|---|---|---|
| `ADNQ26Q` | Poids de plans corrigés + `test_color_contrast(2,…)` | Le texte est-il lisible sur les 30 images, en particulier le morceau 3 (le pire cas : texte noir sur fond noir) ? |
| `ADNQ26R` | `Q` + correctif mémoire (une seule origine `log%`) | La musique se joue-t-elle ? Le programme démarre-t-il toujours ? |
| `ADNQ26S` | `R` + lecture correcte de la palette STE | Couleurs cohérentes (encre 15, bandeau 0) ✅, mais glyphes toujours troués. |
| `ADNQ26T` | `S` + `recolor_font` sans appel de FUNCTION | ✅ **Testé : glyphes nets, texte lisible.** |
| `ADNQ26U` | modifications utilisateur (`$m 450000`, crédits) | ✅ compilé |
| `ADNQ26V` | `U` + fondus STE + bornage + `@tatouche` hors boucles internes | ⚠️ Testé : couleurs correctes, **mais flash toujours présent**. |
| `ADNQ26X` | `V` + fondu simultané des 3 composantes | ⚠️ Testé : plus doux, mais lent au début / rapide à la fin, et flash résiduel sur certaines images. |
| `ADNQ26Y` | `X` + arrondi de la rampe + registre 0 monté après `pixelisation` | ✅ à valider |
| `ADNQ26Z` | `Y` + `pixdraw` par plans de bits, durée pilotée, rideau de texte global | ⚠️ Testé : 5 niveaux tenus 24 s — **trop peu d'étapes, rien ne bouge pendant longtemps**. |
| `ADNQ27A` | `Z` + 9 niveaux (paliers rectangulaires) × 8 bandes = 72 étapes | ✅ Testé : bon effet, mais dessin encore lent et pas assez d'étapes. |
| `ADNQ27B` | `A` + transformation au mot, réplication par doublement, 20 bandes = 180 étapes | ✅ à valider |
| `ADNQ27C` | `B` + `scrambling(duree&)` et alternance des deux effets par morceau | ✅ à valider |
| `ADNQ27D` | `C` + `dissolve.to.pic` : l'espace révèle en 2 s au lieu d'un coup | ✅ à valider |
| `ADNQ27E` | `D` + `dezoom.rc` restauré, alternance à 3 effets | ✅ Testé, fonctionne. |
| `ADNQ27F` | `E` + `random_pixels` réécrite, alternance à **4 effets** | ✅ Testé, fonctionne. |
| `ADNQ27G` | `F` + échelle de `dezoom.rc` corrigée (9 paliers utiles) | **L'effet continue-t-il de progresser jusqu'au bout ?** |

Si `Q` échoue, **rebaser `R`** sur la version qui tourne au lieu de le tester tel quel.

### Pistes non explorées

| Piste | Réserve |
|---|---|
| Filet de sécurité : si l'écart de luminosité encre/registre 0 est trop faible, forcer noir pur / blanc pur **dans `pal%` uniquement** | Code déjà écrit et testé une fois (voir `continue.md` §9.2). Effet de bord : modifie aussi les pixels de l'image qui utilisent ce registre. |
| Affichage en **mode transparent** (`RC_COPY` mode 7 = `s OR d`) | En couleur indexée, un OR donne `fond OR encre` → couleurs imprévisibles. À étudier sérieusement. |
| Recolorer aussi le **fond** des cellules puis dessiner en mode 3 (replace) | Contrôle total des deux couleurs, **mais** supprime l'effet de rideau progressif qui repose sur masque + AND. |
| Couleur de fond du texte variable / aléatoire | Non explorée. |

## Dette technique

| Sujet | Note |
|---|---|
| Instrumentation `chk$` | Encore présente (jalons du flux principal et de `set_text_colors`). À retirer une fois stable. |
| `ALERT` de debug dans `bye` | S'affiche même en sortie normale. À retirer. |
| `pal.order.by.lum` | **Code mort** : `ord.pal&()` est calculé mais jamais lu. Le tri est de plus incorrect (les deux affectations sont à l'intérieur de la boucle interne au lieu d'être après) et l'indexation part du registre 1 tout en stockant l'indice 0. À supprimer ou réécrire si le besoin réapparaît. |
| Restauration écran dans `bye` | `XBIOS(5,L:oldlog%,…)` absent — n'était pas dans la base qui tourne. |
| `TEST1.LST` | À supprimer (« line too long » au Merge, jamais expliqué). |
| Lenteur en interprété | Normale (`recolor_font` ≈ 5000 itérations). À n'évaluer que sur la version compilée. |
