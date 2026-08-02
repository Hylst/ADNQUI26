# Changelog — ADN QUIZZ

Historique des modifications, du plus ancien au plus récent.
Chaque entrée correspond à un commit ; `git show <hash>` en donne le détail complet.

| Date | Commit | Modification |
|---|---|---|
| 2026-08-01 | `aca2f15` | Snapshot before debugging ADNQU26Q bus error |
| 2026-08-01 | `bee167c` | Corrige les poids de plans dans cptst/cpset + documentation du projet |
| 2026-08-01 | `edff5b0` | Prepare le correctif memoire dans ADNQU26R.LST (cycle suivant) |
| 2026-08-01 | `54d0937` | Applique la regle 8.3 de GEMDOS aux fichiers du projet |
| 2026-08-01 | `01c0a5b` | Corrige la lecture de palette STE dans test_color_contrast (ADNQ26S) |
| 2026-08-01 | `a27d0d9` | Corrige la description de FONT.INL dans FEATURES.md |
| 2026-08-01 | `b43f803` | Identifie et corrige la fragmentation des glyphes (ADNQ26T) |
| 2026-08-01 | `e5afca1` | Supprime la multiplication dans recolor_font, ajoute todo.md |
| 2026-08-01 | `25294d1` | Corrige les fondus de palette et allege pixelisation (ADNQ26V) |
| 2026-08-01 | `545877e` | Remplace le balayage de canaux des fondus par une rampe simultanee (ADNQ26X) |
| 2026-08-01 | `c8e1af6` | Supprime le flash residuel et regularise la rampe des fondus (ADNQ26Y) |
| 2026-08-01 | `57a5c8f` | Revelation pilotee en duree et rideau de texte global (ADNQ26Z) |
| 2026-08-01 | `5942d84` | Porte la revelation a 72 etapes visibles (ADNQ27A) |
| 2026-08-02 | `4b8e1d4` | Double la vitesse de pixdraw et le nombre d'etapes (ADNQ27B) |
| 2026-08-02 | `0949e5a` | Audit verifie des effets de revelation disponibles |
| 2026-08-02 | `259533d` | Chantier 1 : scrambling pilote en duree et alternance des effets (ADNQ27C) |
| 2026-08-02 | `c9e0ccb` | Fin progressive commune : dissolve.to.pic (ADNQ27D) |
| 2026-08-02 | `c2b8f99` | Chantier 2 : restaure l'ancien dezoom RC_COPY (ADNQ27E) |
| 2026-08-02 | `5a4c2ee` | Chantier 3 : reecrit random_pixels et complete l'alternance a 4 effets |
| 2026-08-03 | `3e07de0` | Corrige l'echelle de dezoom.rc : neuf paliers utiles (ADNQ27G) |
| 2026-08-03 | `1d61db9` | Corrige le clignotement de screen.opening |
| 2026-08-03 | `bf89991` | 8.1 : ondulation du texte artiste / titre |
| 2026-08-03 | `d5181f0` | Efface log% apres MALLOC et repartit l'attente de dezoom.rc |
| 2026-08-03 | `2fe5a69` | Documente les points tranches et ignore GFX_WIP / OLD |

---

## Repères

- **`aca2f15`** — point de départ, avant le débogage du contraste de la fonte.
- **`bee167c`** — correction des poids de plans dans `cptst`/`cpset` : c'est elle qui a
  débloqué la lisibilité du texte.
- **`01c0a5b`** — découverte que les images sont au format **STE 4096 couleurs** et non ST.
- **`b43f803`** — fin de la fragmentation des glyphes (suppression de l'appel de FUNCTION
  dans la boucle de `recolor_font`).
- **`545877e`** — les fondus deviennent une vraie rampe de luminosité au lieu d'un
  balayage de canaux.
- **`5a4c2ee`** — les quatre effets de révélation partagent enfin la même interface.
- **À partir de `1d61db9`** — le développement se fait sur le seul `ADNQ27G.LST`,
  avec un commit par étape pour permettre un retour arrière.
