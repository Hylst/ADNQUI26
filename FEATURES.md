# Fonctionnalités — ADN QUIZZ

Légende : ✅ fonctionne · ⚠️ fonctionne avec réserve · 🔬 modifié, non testé · ❌ en panne · ⬜ non implémenté

## Déroulé du quiz

| Fonctionnalité | État | Note |
|---|---|---|
| Enchaînement des 30 morceaux | ✅ | |
| Lecture musique SNDH | ⚠️ | « Pas de musique » signalé lors d'un test, jamais investigué. Vérifier d'abord la config audio de Steem SSE (YM/DMA, volume). **Piste sérieuse** : le début de `sndhplay%` (186 octets) peut être écrasé par la fin de `pic%` — correctif dans `ADNQ26R.LST`, cf. `STRUCT.md`. |
| Apparition progressive de l'image (pixellisation, scrambling) | ✅ | |
| Affichage du texte réponse (artiste / titre) | ⚠️ | Lisible seulement si la palette de l'image s'y prête — c'est le sujet ouvert ci-dessous. |
| Contrôle clavier (Espace / Entrée / Backspace) | ✅ | |
| Fondus de palette (`fadeon` / `fadeoff`) | ✅ | |
| Écran d'intro, écran final « fireworks » | ✅ | |

## Le sujet ouvert : contraste du texte

**Le problème.** `DATA\FONT.INL` est une fonte bitmap 8×8 **monochrome**, dessinée
uniquement avec l'index **15**. Chaque `.PI1` embarque sa propre palette, donc le
registre 15 a un RVB différent à chaque image : parfois lisible, parfois
illisible (typiquement sombre sur sombre).

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
| `recolor_font` repeint l'encre | ⚠️ | Utilise `cptst` (une FUNCTION) → **risque `ERR 15`**. Le contournement testé (recopier la logique dans la PROCEDURE) a été perdu lors du retour à la base P ; à réappliquer si l'erreur revient. |
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

D'où le fait que le problème soit apparu **au moment où la recoloration a été
introduite** : la fonte d'origine était en 15, seule valeur usuelle invariante.

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

### Fragmentation des glyphes — cause NON identifiée

Symptôme : sur l'écran d'intro le texte est net, mais dès le 1ᵉʳ morceau les
lettres apparaissent trouées. **Quatre hypothèses ont été vérifiées et écartées :**

| Hypothèse | Verdict |
|---|---|
| Le masque du rideau serait incomplet | **Faux.** Simulé : les états 6 et 7 couvrent 64/64 pixels. |
| La correction des poids de plans | **Impossible.** Elle ne change qu'une couleur uniforme, elle ne peut pas trouer un glyphe. |
| `stab` alterne les tampons, l'un serait en retard | **Faux.** Un tampon finit à l'état 6, l'autre à l'état 7 — les deux sont complets. |
| `choix 2` au lieu de `choix 0` | **Hors de cause pour l'image 01** : les deux donnent 12. |

Constat utile : `FONT.INL` est en **couleur 2** et ne contient que 2 couleurs
(0 et 2). `continue.md` §2 (« index 15 ») et le commentaire du `DATA`
(« 4 colours ») sont **tous deux faux** — vérifié en décodant le fichier.

**Prochaine expérience** : neutraliser `recolor_font` pour les morceaux tout en
gardant `set_text_colors` pour les couleurs de bandeau. Texte net en couleur 2
⇒ le coupable est `recolor_font` ; texte toujours troué ⇒ le problème est dans
la chaîne d'affichage et `recolor_font` est hors de cause.

### File d'attente des cycles de test

| Version | Contenu | À vérifier |
|---|---|---|
| `ADNQ26Q` | Poids de plans corrigés + `test_color_contrast(2,…)` | Le texte est-il lisible sur les 30 images, en particulier le morceau 3 (le pire cas : texte noir sur fond noir) ? |
| `ADNQ26R` | `Q` + correctif mémoire (une seule origine `log%`) | La musique se joue-t-elle ? Le programme démarre-t-il toujours ? |
| `ADNQ26S` | `R` + lecture correcte de la palette STE | Les couleurs choisies sont-elles enfin cohérentes (encre 15, bandeau 0) ? |

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
