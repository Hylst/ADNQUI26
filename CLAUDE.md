# CLAUDE.md — ADN QUIZZ

**Lire [`AGENTS.md`](AGENTS.md) en premier.** Il contient la méthode de travail et
les pièges vérifiés, et il fait autorité. Ce fichier n'ajoute que ce qui est
spécifique à Claude Code.

## Base documentaire interrogeable

Un corpus de 41 ouvrages Atari ST (13 000 pages, 22 343 passages) est indexé dans
`D:\0CODE\ClaudeCode\atari\`. **L'interroger avant toute affirmation technique** —
syntaxe GFA, adresse de registre, opcode 68000, appel GEM.

```bash
cd D:/0CODE/ClaudeCode/atari

python tools/kb.py lookup PSET --full        # syntaxe exacte d'un mot-cle GFA
python tools/kb.py lookup '$FF8240' --full   # format de bits d'un registre
python tools/kb.py lookup movem              # mnemonique 68000
python tools/kb.py search "eviter le scintillement" -k 6
python tools/kb.py docs                      # les 41 documents indexes
```

Chaque résultat cite `document p.N`, ce qui permet de remonter au PDF d'origine
dans `docs_pdf/` pour vérifier.

**Ce que la base sait faire** : tables de registres, syntaxe des commandes,
appels système. Réponse en quelques lignes au lieu de plusieurs dizaines de
milliers de tokens.

**Ce qu'elle ne sait pas faire** : les schémas et figures, détruits par l'OCR.
Pour un diagramme, ouvrir le PDF directement.

**Si la base ne répond pas, le dire** plutôt que de supposer.

Les manuels GFA bruts restent la référence de dernier recours, cf. `AGENTS.md`.

## Édition du `.LST`

Le `.LST` est **UTF-8 avec CRLF**, mais destiné à un éditeur Atari qui ne lit pas
l'UTF-8 multi-octets. N'écrire que de l'**ASCII pur** dans le code et les
commentaires ajoutés — pas d'accent, pas de tiret cadratin, pas d'apostrophe
typographique.

Après chaque édition, vérifier :

```bash
cd "/d/0CODE/ATARI24/0GFA/2026/ADNQUI26"
diff ADNQU26Q.LST.avant-<motif> ADNQU26Q.LST    # le diff ne contient QUE l'intention
python -X utf8 -c "b=open('ADNQU26Q.LST','rb').read(); \
print('LF orphelins',b.count(b'\n')-b.count(b'\r\n')); \
print('octets >127',sum(1 for x in b if x>127))"
```

Le compte d'octets > 127 doit rester à **41** (dégâts antérieurs, à ne pas aggraver).
Les `>` en tête de certaines lignes `PROCEDURE` sont des marqueurs de pliage GFA
légitimes : ne pas les retirer.

## Rappel

L'utilisateur édite en parallèle dans l'éditeur GFA et recompile. **Relire les
fichiers avant d'agir**, ne pas se fier à une lecture antérieure dans la même
session — plusieurs versions ont déjà changé en cours de travail.
