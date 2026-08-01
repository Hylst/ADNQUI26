# Structure — ADN QUIZZ

## Fichiers du projet

| Fichier | Rôle |
|---|---|
| `ADNQ26WO.LST/.GFA/.PRG` | **Référence qui tourne.** Identique à `ADNQU26P`. Ne pas écraser. |
| `ADNQU26Q.LST/.GFA/.PRG` | Développement — **cycle en cours** : plans de bits + `choix 2`. |
| `ADNQU26R.LST` | `Q` + correctif mémoire — **cycle suivant**. `.GFA`/`.PRG` à générer. |
| `ADNQUI26.*`, `…26N`, `…26O`, `…26P`, `…26W` | Versions antérieures, conservées. |
| `*.BAK` | Sauvegardes de l'éditeur GFA. |
| `GFABASIC.PRG` | Copie locale de l'interpréteur. |
| `DATA\` | Ressources (fonte, images, musiques). |
| `AUTO\` | Dossier de démarrage automatique. |
| `TEST1.LST` | Fichier de test — provoque un « line too long » jamais expliqué. À supprimer. |

Seul le `.LST` est éditable directement. Voir `README.md` pour le cycle de build.

## Carte mémoire

Le programme alloue **un seul bloc** puis y découpe tous ses tampons.

```gfa
$m 400000                        ! TPA du programme compile
RESERVE 400000                   ! libere le surplus vers GEMDOS  (PORTEUR, cf. AGENTS.md)
mem_block%=MALLOC(total_size%)   ! ~488 Ko, pris sur ce qui vient d'etre libere
log%=(mem_block%+255) AND &HFFFFFF00
```

`total_size%` = `32256 + 32 + 32 + 32 + (22+8)*160 + 32034` = **69186**, plus la
taille des fichiers de base, plus 200000 pour la musique.

### Chaîne A — tampons écran, basée sur `log%` (alignée sur 256)

| Tampon | Adresse | Taille | Rôle |
|---|---|---|---|
| `log%` | `align256(mem_block%)` | 32256 | écran logique (double buffer) |
| `pal%` | `log% + 32256` | 32 | palette de l'image courante |
| `stzpal%` | `+ 32` | 32 | palette noire (fondu) |
| `savpal%` | `+ 32` | 32 | palette sauvegardée |
| `lowborderm%` | `+ 32` | 22×160 | lowborder / tampon d'effacement |
| `fmask%` | `+ 3520` | 8×160 | masque du rideau d'apparition |
| `pic%` | `+ 1280` | 32034 | image `.PI1` courante |

### Chaîne B — fichiers, basée sur `mem_block%`

| Tampon | Adresse | Taille mesurée |
|---|---|---|
| `sndhplay%` | `mem_block% + cumul_size%(0)` | 186 |
| `sp3%` | `mem_block% + cumul_size%(1)` | 1602 |
| `font%` | `mem_block% + cumul_size%(2)` | 2560 (320×16) |
| `tune%` | `font% + 16*160` | 200000 |

> ### ⚠️ Point d'attention non résolu — double origine
>
> La chaîne A part de `log%` (**arrondi** à 256), la chaîne B de `mem_block%`
> (**non arrondi**). Or `cumul_size%(0)` vaut exactement 69186, soit la longueur
> de la chaîne A. Les deux chaînes ne sont donc jointives que si `mem_block%` est
> déjà aligné sur 256.
>
> Sinon, la fin de `pic%` déborde de `pad` octets (0 à 255) sur le **début de
> `sndhplay%`**, qui ne fait que 186 octets — donc sur ses points d'entrée.
>
> **Statut : hypothèse non testée.** Elle expliquerait le symptôme « pas de
> musique » resté en suspens.
>
> **Le correctif existe dans `ADNQU26R.LST`** (une seule origine, `log%` partout) :
>
> ```gfa
> mem_block%=MALLOC(total_size%+256)   ! marge pour l'arrondi
> log%=(mem_block%+255) AND &HFFFFFF00
> sndhplay%=log%+cumul_size%(0)        ! log% et non mem_block%
> sp3%     =log%+cumul_size%(1)
> font%    =log%+cumul_size%(2)
> ```
>
> Les deux chaînes deviennent jointives **par construction**, quelle que soit
> l'adresse rendue par `MALLOC`. Le `+256` est nécessaire : la dernière adresse
> utilisée est `log%+total_size%`, jusqu'à 255 octets au-delà de
> `mem_block%+total_size%`.
>
> ⚠️ Ce `+256` augmente très légèrement la demande d'allocation. Si le `MALLOC`
> était déjà au ras de la mémoire libérée par `RESERVE`, il pourrait échouer →
> `EDIT` → écran noir. Peu probable, mais c'est le premier symptôme à
> reconnaître si `R` ne démarre pas.

Un chevauchement du même genre a déjà été corrigé : `tune%` valait `pic%+32034`,
qui tombait exactement sur `sndhplay%` — chaque chargement de musique (38 à 75 Ko)
écrasait le lecteur SNDH et la fonte.

## Organisation du code

Flux principal (en tête du `.LST`, chaque étape jalonnée par `chk$`) :

```
hello  →  get_slide_filenames  →  base_files_load  →  clearpal  →  pmul160
       →  prepfont  →  set_text_colors  →  sndh  →  screen.opening
       →  intro_quizz  →  quizz  →  bye
```

### Procédures notables

| Nom | Rôle |
|---|---|
| `hello` | Allocation mémoire, sauvegardes système, `DIM` groupés en amont |
| `prepfont` | Construit le masque du rideau (`COLOR 1` — cf. `AGENTS.md`) |
| `display.bitmap.txt` | Affiche un texte via les deux `RC_COPY` (masque replace + fonte AND) |
| `test_color_contrast` | Choisit un registre : `0` le plus clair, `1` le plus sombre, `2` le plus contrasté avec le registre 0 |
| `recolor_font` | Repeint l'encre de `font%` dans la couleur donnée (~5000 itérations) |
| `set_text_colors` | Calcule `ink_col&` + `box_col&` puis appelle `recolor_font` |
| `area.lines.erase` | Efface le rectangle sous le texte, en `COLOR vdicol|(box_col&)` |
| `cptst` / `cpset` | Lecture / écriture d'un pixel en mémoire, 4 plans |
| `pal.order.by.lum` | Tri de palette par luminance — **code mort**, jamais lu (voir `FEATURES.md`) |
| `vdicol` | Charge la table de conversion registre → index VDI |
| `fadeon` / `fadeoff` | Fondus de palette |

### Variables globales de couleur

| Variable | Contenu |
|---|---|
| `ink_col&` | Numéro de **registre matériel** de l'encre du texte |
| `box_col&` | Numéro de **registre matériel** du fond du rectangle |
| `vdicol|(n)` | Table : registre matériel → index VDI |
| `pal%` | Palette de l'image courante (16 mots) |

Toute variable contenant un numéro de registre doit passer par `vdicol|()` avant
d'être donnée à `COLOR`, `DEFFILL`, `DEFTEXT`.
