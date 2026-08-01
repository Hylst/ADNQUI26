# ADN QUIZZ 2026

Quiz musical pour **Atari ST**, écrit en **GFA BASIC 3** et compilé en `.PRG`.

Pour chacun des 30 morceaux :

1. la musique seule (fichier `.SND`, format SNDH) ;
2. l'image apparaît progressivement (pixellisation puis scrambling) ;
3. le texte de la réponse s'affiche par-dessus — ligne 1 l'artiste, ligne 2 le titre.

**Contraintes** : 320×200 en 16 couleurs indexées (basse résolution ST), machine
peu puissante, mémoire limitée. Chaque image `.PI1` embarque **sa propre palette**.

## Cycle de travail

```
.LST  ──Merge──►  éditeur GFA  ──Save──►  .GFA  ──compil──►  .PRG  ──►  Steem SSE
```

Le `.LST` (ASCII) est **le seul fichier éditable directement**. Le `.GFA` est le
format tokenisé natif, le `.PRG` l'exécutable. Après toute modification du `.LST`,
il faut refaire le cycle complet : les trois fichiers ne sont plus cohérents entre eux.

| Élément | Emplacement |
|---|---|
| Éditeur / interpréteur / compilateur / linkeur GFA | `D:\0CODE\ATARI24\0GFA\2026\GFA36TTE\` (`menu.prg` bascule de l'un à l'autre) |
| ROM TOS | `D:\0CODE\ATARI24\Steem.SSE.4.0.0.Win64\TOS162FR.img` (STE d'origine) |
| Émulateur Steem SSE | `D:\0CODE\ATARI24\Steem.SSE.4.0.0.Win64\Steem64 4.exe` |
| Émulateur Hatari | `D:\0CODE\ATARI24\hatari-2.6.1_windows64\hatari.exe` |

## Versions

Le projet avance par **fichiers suffixés** (`ADNQUI26` → `…26N` → `…26O` → `…26P` → `…26Q`).
Un dépôt git est initialisé sur `main`.

- **`ADNQ26WO.*`** — copie de référence **qui tourne** (identique à `ADNQU26P`, MD5 vérifié).
  Ne jamais l'écraser : c'est le point de retour.
- **`ADNQU26Q.*`** — **à tester maintenant** : correction des poids de plans de bits
  et `test_color_contrast(2,…)`.
- **`ADNQU26R.LST`** — **cycle suivant** : `Q` + correctif mémoire (double origine
  des tampons). `.GFA` / `.PRG` restent à générer par le cycle de build.

Les versions sont volontairement empilées une par cycle de test : si `Q` échoue,
`R` est à rebaser sur ce qui tourne plutôt qu'à tester tel quel.

## Documentation

| Fichier | Contenu |
|---|---|
| [`AGENTS.md`](AGENTS.md) | **À lire en premier par tout agent.** Méthode de travail et pièges vérifiés. |
| [`CLAUDE.md`](CLAUDE.md) | Spécificités Claude Code (base documentaire interrogeable). |
| [`STRUCTURE.md`](STRUCTURE.md) | Carte mémoire, organisation du code, fichiers de données. |
| [`FEATURES.md`](FEATURES.md) | Fonctionnalités et leur état réel. |
| [`continue.md`](continue.md) | Journal de la session de débogage du 2026-08-01 (historique détaillé). |

## État actuel

Le programme **fonctionne** dans sa version `ADNQ26WO` / `ADNQU26P`.

Le sujet ouvert est le **contraste du texte** : la fonte est monochrome (index 15)
alors que chaque image a sa propre palette, d'où un texte parfois illisible.
Voir `FEATURES.md` pour le détail et `AGENTS.md` §Pièges avant toute modification.
