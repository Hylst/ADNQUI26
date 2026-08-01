# ADN QUIZZ (Atari ST / GFA Basic 3) — Reprise de travail

> Document de passation. Rédigé le 2026-08-01 à la fin d'une session de débogage.
> Objectif : permettre à un autre agent de reprendre sans refaire les erreurs déjà commises.

---

## 1. Le projet

**ADN QUIZZ** : quiz musical pour Atari ST, écrit en **GFA Basic 3**, compilé en `.PRG`.

Déroulé pour chacun des 30 morceaux :
1. La musique seule (fichier `.SND`, format SNDH)
2. L'image apparaît progressivement (effet de pixellisation, puis scrambling)
3. Le texte de la réponse s'affiche par-dessus l'image : **ligne 1 = artiste, ligne 2 = titre**

Contraintes matérielles : **320×200, 16 couleurs indexées** (basse résolution ST), machine peu puissante, mémoire limitée.

### Dossier de travail

```
D:\0CODE\ATARI24\0GFA\2026\ADNQUI26\
```

Un dépôt **git** y est initialisé (branche `main`, 1 commit : « Snapshot before debugging ADNQU26Q bus error »).

### Fichiers

| Extension | Rôle |
|---|---|
| `.LST` | Source **texte ASCII** — c'est le seul fichier éditable par un agent |
| `.GFA` | Format **tokenisé natif** de l'éditeur GFA (binaire, non éditable) |
| `.PRG` | Exécutable compilé |

**Cycle de travail de l'utilisateur** (important à comprendre) :
`.LST` → **Merge** dans l'éditeur GFA → Save (`.GFA`) → compilation (`.PRG`) → test sous **Steem SSE**

L'utilisateur utilise bien **Merge** (import ASCII), confirmé.

### Ressources de documentation disponibles dans `D:\0CODE\OpenCode\Pixel_Bitmap_Gen\`

- `GFABasic.HYP.html` — manuel GFA Basic complet (le plus fiable, très détaillé)
- `GFAXPERT2.txt` — « Your Second GFA-BASIC 3.0 Manual » (Han Kempen) — contient la table VDI/matériel
- `gfamanual.txt` — manuel de référence

**Ces documents ont été décisifs plusieurs fois. Les consulter avant toute hypothèse.**

---

## 2. Le problème à résoudre

`DATA\FONT.INL` est une fonte bitmap 8×8 **monochrome**, dessinée uniquement avec **l'index de couleur 15**.

Chaque image `.PI1` embarque **sa propre palette**. Donc le registre matériel 15 a un RVB différent à chaque image : parfois le texte contraste bien, parfois il est **illisible** (typiquement sombre sur sombre).

**But : garantir un texte lisible sur les 30 images, sans modifier les images** (elles sont figées ; sacrifier 2 couleurs sur 16 dégraderait le rendu en palette indexée — option explicitement écartée par l'utilisateur).

---

## 3. Découvertes techniques (vérifiées dans le manuel)

### 3.1 Comment le texte est réellement dessiné

Dans `display.bitmap.txt`, chaque caractère est composé en **deux** `RC_COPY` :

```gfa
RC_COPY fmask%,m.appear&,0,8,8 TO log%,x,y        ! pas de mode => mode 3 (replace)
RC_COPY font%,...,char.width&,char.height& TO log%,x,y,1   ! mode 1 = s AND d
```

Le mode de `RC_COPY` est une **opération logique bit à bit sur les 4 plans**, valeurs 0 à 15
(manuel, section `RC_COPY` → renvoie à `PUT`) :

| Mode | Effet |
|---|---|
| 0 | tout à 0 |
| **1** | **s AND d** ← utilisé ici |
| **3** | **replace (défaut)** |
| 6 | XOR |
| 7 | transparent (s OR d) |
| 13 | transparent inverse |
| 15 | tout à 1 |

Donc le résultat est : **`fonte AND masque`**.

### 3.2 Conséquence majeure : le fond de chaque lettre est TOUJOURS le registre 0

Le masque (« rideau » d'apparition, construit dans `prepfont`) est tracé avec `COLOR 1`.
Là où le masque est allumé → les 4 plans sont à 1 → le AND **préserve la couleur de la fonte**.
Là où il est éteint → **0**.

⇒ À l'intérieur de chaque cellule 8×8, le fond est **systématiquement le registre matériel 0** de l'image courante.
C'est ce qui produit le « rectangle sombre » visible derrière le texte sur les captures.

**C'est donc avec le registre 0 que l'encre doit contraster** — pas avec l'image en général.

### 3.3 ⚠️ Index VDI ≠ registre matériel (piège majeur)

`COLOR`, `DEFFILL`, `DEFTEXT`… attendent un **index VDI**, pas un numéro de registre matériel.
Table (source : `GFAXPERT2.txt`, section *SETCOLOR and VSETCOLOR*) :

```
index VDI   : 0  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15
registre HW : 0 15  1  2  4  6  3  5  7  8  9 10 12 14 11 13
```

Le programme contient déjà le tableau inverse `vdicol|()` (`DATA 0,2,3,6,4,7,5,8,9,10,11,14,12,15,13,1`),
soit `vdicol|(registre_HW) = index_VDI`. Vérification : `vdicol|(15) = 1` ✔

Conséquences pratiques :
- `COLOR 1` écrit le registre matériel **15** = `1111` (les 4 plans) → **correct** pour le masque
- `COLOR 15` écrirait le registre **13** = `1101` → **perdrait un plan** (erreur commise puis annulée)
- Toute variable contenant un **numéro de registre** doit passer par `vdicol|()` avant `COLOR`/`DEFFILL`

---

## 4. Solution mise en place

Trois procédures, appelées **après chaque `ldpic`** (intro, chaque morceau dans `quizz`, image finale « fireworks ») :

| Procédure | Rôle |
|---|---|
| `test_color_contrast(choix&, VAR col_number&)` | Préexistante mais **jamais appelée** avant. `choix 0` = plus claire, `choix 1` = plus sombre, `choix 2` = **plus contrastée avec le registre 0**. Modifiée pour lire la palette dans `pal%` (`CARD{pal%+i+i}`) au lieu de `XBIOS(7,...)` : plus rapide et indépendant du timing du fade. |
| `recolor_font(couleur&)` | **Nouvelle.** Repeint tous les pixels d'encre du buffer `font%` (320×16) dans la couleur donnée. ~5000 itérations, une seule fois par morceau. |
| `set_text_colors` | **Nouvelle.** Calcule `ink_col&` + `box_col&`, puis appelle `recolor_font(ink_col&)`. |

`area.lines.erase` (qui efface le rectangle sous le texte) utilise désormais `box_col&` au lieu de `COLOR 0` figé,
**avec conversion** : `COLOR vdicol|(box_col&)` / `DEFFILL vdicol|(box_col&)`.

---

## 5. Bugs trouvés et corrigés (présents dans le fichier actuel)

### 5.1 Chevauchement mémoire — bug réel et sérieux ✔ CORRIGÉ

`tune%` était calculé `pic%+32034`, adresse qui tombait **exactement sur `sndhplay%`**.
Or `sndhplay%` + `sp3%` + `font%` ne pèsent que **4348 octets** (186 + 1602 + 2560 vérifiés),
alors que chaque `BLOAD` de musique charge **38 à 75 Ko** au même endroit.
⇒ Le lecteur SNDH et la fonte étaient écrasés à chaque changement de morceau.

Correction :
- `tune%=font%+16*160` (juste après la fonte)
- le tampon musique de 200000 octets est ajouté à `total_size%` **après** la boucle de calcul des fichiers de base

### 5.2 Conversion VDI manquante dans `area.lines.erase` ✔ CORRIGÉ

Voir §3.3.

---

## 6. ⚠️ Pièges confirmés — NE PAS refaire ces erreurs

### 6.1 `RESERVE` est PORTEUR — ne jamais le désactiver en compilé

Le manuel dit « *should not be used in compiled programs* », **et pourtant il est indispensable ici.**

Mécanisme :
```
$m 400000              ! donne 400 Ko au programme compilé
RESERVE 400000         ! libère le surplus vers GEMDOS
MALLOC(total_size%)    ! ~488 Ko, ne peut venir QUE de ce qui vient d'être libéré
```
En entourant `RESERVE` d'un `IF NOT compiled!`, le `MALLOC` échoue → `mem_block%<=0` → le code tombe dans `EDIT`
→ **sortie immédiate, écran noir**.

**Erreur commise deux fois dans la session.** Un commentaire d'avertissement est en place dans le fichier.

### 6.2 `ERR 15 Array not dimensioned` dans `recolor_font`

`cptst` est une **FUNCTION** (pas une PROCEDURE) et accède au tableau global `t160&()`.
C'était le premier endroit du programme où une FUNCTION touchait ce tableau
(`cptst`/`cpset` n'avaient jamais été réellement exécutées avant).
Résultat : `ERR 15` par intermittence, exactement à l'étape `set_text_colors:recolor`.

Le manuel signale par ailleurs des bizarreries de portée dans les FUNCTION
(« *DIM should not be used within a FUNCTION. It seems to clobber any LOCAL variables* »).

**Contournement qui a fonctionné** : recopier la logique de `cptst`/`cpset` **directement dans** `recolor_font`
(une PROCEDURE), sans aucun appel de FUNCTION.

> ⚠️ **État actuel : cette correction n'est PAS dans le fichier.** `recolor_font` a été ramenée à la version
> `cptst` lors du retour à la base P. Si `ERR 15` réapparaît, réappliquer l'inlining
> (la version inline est retrouvable dans l'historique de la conversation, ou à réécrire depuis `cptst`/`cpset`).

### 6.3 Hypothèses fausses — ne pas les reprendre

| Hypothèse | Verdict |
|---|---|
| `RESERVE` doit être désactivé en compilé (le manuel le dit) | **FAUX** — casse le `MALLOC`, cf. §6.1 |
| `COLOR 15` pour le masque du rideau | **FAUX** — VDI 15 = registre 13, cf. §3.3. `COLOR 1` est correct |
| `ADNQU26P.LST` serait un export périmé, différent du `.GFA` qui tourne | **FAUX** — MD5 identique à `ADNQ26WO.LST` |
| Guillemet non fermé / longueur de ligne = cause de l'écran noir | **Non démontré.** Un guillemet `"` non fermé dans un commentaire a bien existé dans une version (à éviter par principe), mais rien ne prouve que c'était la cause |
| `font.chars$` contiendrait `\` au lieu de `/` | **FAUX** — artefact d'affichage d'un outil. Le fichier contient bien `/` |

### 6.4 Règles de formatage des `.LST` (par prudence)

- **ASCII uniquement** dans tout ajout (le fichier contient 41 octets non-ASCII préexistants — accents dans de vieux commentaires — ne pas en ajouter)
- **CRLF** partout, pas de BOM
- **Pas de guillemet `"` dans les commentaires** (et jamais de guillemet non apparié sur une ligne)
- Lignes raisonnablement courtes (le fichier monte à 162 caractères sans problème)

Contrôle rapide en PowerShell :
```powershell
$q="D:\0CODE\ATARI24\0GFA\2026\ADNQUI26\ADNQU26Q.LST"
$l=Get-Content $q
for($i=0;$i -lt $l.Count;$i++){ if((([regex]::Matches($l[$i],'"')).Count % 2) -ne 0){ "IMPAIR $($i+1): $($l[$i])" } }
```
> Note : **une** ligne impaire est normale et préexistante (commentaire de la section DATA, ligne ~1274).

---

## 7. Méthode de débogage qui a fonctionné

L'itération est lente (édition → Merge → Save → compilation → test sous émulateur). Ce qui a payé :

1. **Variable `chk$`** positionnée avant chaque étape, affichée en cas d'erreur → a permis de localiser `ERR 15` précisément.
2. **`ALERT` dans `bye`** affichant `ERR`, `ERR$(ERR)` et `chk$`.
   ⚠️ `bye` doit d'abord **restaurer le mode utilisateur** (`GEMDOS(32,...)`) avant l'`ALERT`, sinon rien ne s'affiche.
3. **Ne changer qu'une chose à la fois**, et comparer au fichier de référence qui tourne :
   ```powershell
   Compare-Object (Get-Content ADNQ26WO.LST) (Get-Content ADNQU26Q.LST)
   ```

### Limites constatées
- Sans `ON ERROR GOSUB`, une erreur en mode superviseur + écran custom ne s'affiche pas → écran noir figé, reset obligatoire.
- Une **erreur bus (« 2 bombes »)** est une exception matérielle 68000 : elle **contourne** `ON ERROR GOSUB`.
- `ON ERROR` ne réagit **pas** à un blocage (boucle d'attente infinie), seulement aux erreurs.
- `ON BREAK GOSUB bye` est actif : la touche **Undo** peut débloquer certains cas.

---

## 8. État exact du fichier `ADNQU26Q.LST` (audit vérifié)

**Base : `ADNQ26WO.LST` (= `ADNQU26P.LST`, MD5 identique), version qui tourne.**

| Élément | État |
|---|---|
| Correctif mémoire `tune%=font%+16*160` | ✅ présent (l. 103) |
| `ADD total_size%,200000` après la boucle | ✅ présent (l. 79) |
| `COLOR/DEFFILL vdicol|(box_col&)` | ✅ présent (l. 869 / 880) |
| `test_color_contrast` lit `pal%` | ✅ présent (l. 929) |
| `COLOR 1` dans `prepfont` (masque) | ✅ correct (l. 571) |
| `recolor_font` via `cptst` (FUNCTION) | ⚠️ présent (l. 982) — **risque `ERR 15`**, cf. §6.2 |
| `chk$` (instrumentation debug) | ⚠️ présent (l. 11–31, 997–1003) — **à retirer** |
| `ALERT` de debug dans `bye` | ⚠️ présent (l. 290) — **à retirer** (s'affiche même en sortie normale) |
| Restauration écran `XBIOS(5,L:oldlog%,...)` dans `bye` | ❌ absent (n'était pas dans la base qui tourne) |

### Seule différence de code avec la version qui tourne

```gfa
' AVANT (WO) :  test_color_contrast(0,ink_col&)   ! la plus claire dans l'absolu
' APRES (Q)  :  test_color_contrast(2,ink_col&)   ! la plus contrastée avec le registre 0
```

**⚠️ CE CHANGEMENT N'A PAS ENCORE ÉTÉ TESTÉ** au moment d'écrire ce document.

---

## 9. Ce qu'il reste à faire

### Immédiat
1. **Tester `ADNQU26Q.LST`** (Merge → Save → compiler → lancer). Vérifier le contraste, en particulier sur les images sombres (le morceau 3 était le pire cas : texte noir sur fond noir).

### Ensuite, si le contraste reste insuffisant sur les images très sombres
2. **Filet de sécurité** — déjà écrit et testé une fois, mais retiré pendant la chasse aux bugs.
   Principe : si l'écart de luminosité entre l'encre et le registre 0 est trop faible, forcer noir pur / blanc pur
   **uniquement dans `pal%`** (pas sur le matériel, pour que `@fadeon` lise la version corrigée).
   Ne s'applique qu'à l'image concernée, réinitialisé au chargement de la suivante.

```gfa
LOCAL bg_w&,ink_w&,bg_lum,ink_lum
bg_w&=CARD{pal%}
ink_w&=CARD{pal%+ink_col&+ink_col&}
bg_lum=0.3*SHR&((bg_w& AND &HF00),8)+0.59*SHR&((bg_w& AND &HF0),4)+0.11*(bg_w& AND &HF)
ink_lum=0.3*SHR&((ink_w& AND &HF00),8)+0.59*SHR&((ink_w& AND &HF0),4)+0.11*(ink_w& AND &HF)
IF ABS(ink_lum-bg_lum)<3
  CARD{pal%}=&H0000
  CARD{pal%+ink_col&+ink_col&}=&H0777
ENDIF
```
Seuil `3` arbitraire (échelle 0–7), à ajuster.
Effet de bord à surveiller : forcer le registre d'encre en blanc pur modifie aussi les pixels de l'image
qui utilisent cette couleur.

### Pistes proposées par l'utilisateur, non explorées
3. **Affichage du texte en mode transparent** (« l'idéal est qu'il soit collé en mode transparent ») —
   utiliser le mode 7 (`s OR d`) plutôt que le mode 1. Attention : en couleur indexée, un OR donne
   `fond OR encre`, ce qui produit des couleurs imprévisibles. À étudier sérieusement avant de se lancer.
4. **Couleur de fond du texte variable / aléatoire.**
5. Alternative jamais implémentée : recolorer aussi le **fond** des cellules dans `recolor_font`
   (encre → `ink_col&`, fond → `box_col&`) puis dessiner en mode 3 (replace) → contrôle total des deux
   couleurs. **Mais** cela supprimerait l'effet de rideau progressif, qui repose sur le masque + AND.

### Nettoyage une fois stable
6. Retirer `chk$` (l. 11–31, 997–1003) et l'`ALERT` de `bye` (l. 290).
7. Supprimer `TEST1.LST` (fichier de test, provoquait un « line too long » jamais expliqué —
   ses lignes font pourtant 69 caractères au maximum ; ne pas y perdre de temps).

---

## 10. Points en suspens / non élucidés

- **Pas de musique** signalée lors d'un test (« je vois l'écran d'intro, pas de musique »). Jamais investigué.
  Vérifier d'abord la configuration audio de **Steem SSE** (YM/DMA, volume) avant de suspecter le code.
  À noter : le correctif mémoire §5.1 devrait justement avoir réparé le lecteur SNDH, qui était écrasé.
- **Lenteur en mode interprété** : normale (`recolor_font` = ~5000 itérations). À évaluer uniquement
  sur la version **compilée**.
- **« line too long »** au Merge de `TEST1.LST` : jamais expliqué, cf. §9.7.

---

## 11. Rappel de méthode pour l'agent suivant

Le cycle de test est **lent et coûteux** pour l'utilisateur. Par conséquent :

- **Un seul changement à la fois**, systématiquement diffé contre la dernière version qui tourne.
- Toujours **garder une copie de référence qui fonctionne** (`ADNQ26WO.*` aujourd'hui).
- Avant d'affirmer qu'un mécanisme GFA se comporte d'une certaine façon, **le vérifier dans
  `GFABasic.HYP.html` ou `GFAXPERT2.txt`** — plusieurs erreurs de cette session viennent d'hypothèses
  non vérifiées (modes de `RC_COPY`, table VDI, `RESERVE`).
- Attention : le manuel donne des **règles générales** qui peuvent être contredites par l'usage
  délibéré qu'en fait ce programme précis (cas de `RESERVE`, §6.1). Lire le code environnant avant
  de « corriger » ce qui ressemble à une négligence.
