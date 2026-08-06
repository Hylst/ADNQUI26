```
    _    ____  _   _    ___  _   _ ___ ________
   / \  |  _ \| \ | |  / _ \| | | |_ _|__  /__ /
  / _ \ | | | |  \| | | | | | | | || |  / /  / /
 / ___ \| |_| | |\  | | |_| | |_| || | / /_ / /_
/_/   \_\____/|_| \_|  \__\_\\___/|___/____|____|
                                    A T A R I  S T
```

# ADN Quizz 2026

Un blind test musical qui tourne sur **Atari ST**. En **GFA Basic 3**, parce que
oui, on est en 2026 et je code encore en GFA. Assumé.

Le principe : 30 morceaux. Pour chacun, la musique démarre sur un écran noir, puis
l'image se dévoile tout doucement pendant deux minutes — de gros pavés baveux
jusqu'au pixel près. Le temps de laisser les copains ramer. Quand ils sèchent (ou
qu'ils ont trouvé), espace, et l'artiste + le titre s'affichent en ondulant sur
l'image.

À la fin, feu d'artifice et scrolltext. Évidemment. On ne se refait pas.

## Ce qu'il faut pour le faire tourner

- Un **Atari ST** (ou STE, c'est mieux, il y a le blitter)
- Ou un émulateur : je bosse sous **Steem SSE**, ça marche aussi sous **Hatari**
- TOS 1.62 — c'est celui de mon STE, les autres devraient passer

Balancez `ADNQ27G.PRG` et le dossier `DATA` sur une disquette (ou un dossier monté
en disque dur), lancez, et voilà.

L'essentiel se fait à l'**espace** : ça abrège l'effet en cours, ça affiche la
réponse, ça passe au morceau suivant. Backspace pour revenir en arrière si vous
avez la main lourde. À la fin, espace encore et on retourne au bureau GEM
proprement — j'ai mis un moment à obtenir ça, d'ailleurs.

Sur l'écran d'accueil, les **flèches** et **Entrée** ouvrent un petit menu : noter
les joueurs, voir les scores, relire les règles, ou écouter les morceaux à la
carte. Et pendant le quizz, **F1 à F8** donnent le point au joueur qui a trouvé.
Si son nom n'est pas encore enregistré, on vous le demande sur le coup, sans
interrompre le dévoilement de l'image.

## Le contenu

Les musiques sont au format **SNDH**, jouées par une routine assembleur. Ce ne
sont pas des enregistrements : chaque générique a été repris note à note et
réécrit pour le YM2149, trois voies et un générateur de bruit. Merci à **AD** et
**DMA SC** pour ce boulot de fourmi, et à **JESS** pour l'intro.

Les images sont des `.PI1` en 320×200, 16 couleurs. Format STE (4096 couleurs, pas
512 — j'ai perdu du temps là-dessus, la palette est encodée bizarrement sur STE).

La fonte bitmap 8×8 est maison.

## Sous le capot, pour les curieux

Quelques trucs dont je suis assez content :

**La révélation progressive** ne passe pas par le VDI. Je travaille directement sur
les plans de bits : un masque, quelques décalages en `LONG`, et un `BMOVE` pour
répliquer les lignes. L'ancienne version faisait 4500 `RC_COPY` par palier, autant
dire qu'elle ramait. Là c'est 40 itérations par ligne et ça passe crème.

**Quatre effets** alternent d'un morceau à l'autre : dézoom net, dézoom déformé
(l'ancien, en `RC_COPY`, je l'ai gardé parce que son côté baveux a du charme),
lignes mélangées, et blocs dispersés. Tous partagent la même interface de durée,
donc c'est réglable d'une variable.

**Le feu d'artifice** : quatre gerbes simultanées à des stades différents, 16
particules chacune, positions en seizièmes de pixel pour que ça soit fluide sans
toucher au flottant. Effacement ciblé pixel par pixel — pas de `CLS` bourrin.

**Le VU-mètre** de l'écran d'intro lit les registres de volume du YM2149 via
Giaccess. Trois barres, une par voie, posées au fond de l'écran du moniteur qui
est dessiné dans l'image. J'avais écrit un truc similaire en 1993, ça fait
plaisir de ressortir la recette.

**La croche en fil de fer** tourne sur trois axes avec une traînée. Matrice de
rotation en virgule fixe sur 7 bits, calculée une fois par image ; les deux faces
de la tête se déduisent l'une de l'autre par une translation, ce qui économise la
moitié des sommets. Et la traînée ne coûte rien du tout : je garde les points
déjà projetés des images précédentes dans un anneau, il n'y a plus qu'à les
retracer.

**Le scrolltext de l'intro** est monté dans une bande hors écran puis recopié
d'un bloc à chaque image. Ça a l'air anodin ; c'est ce qui a réglé un
clignotement carabiné, parce qu'avec le double écran un tampon sur deux ne
recevait qu'une colonne sur deux.

## Le cycle de build

```
.LST  --Merge-->  editeur GFA  --Save-->  .GFA  --compil-->  .PRG  -->  Steem
```

Le `.LST` c'est l'ASCII, le seul truc éditable au clavier PC. Le `.GFA` c'est le
format tokenisé de l'éditeur, le `.PRG` l'exécutable. Si vous touchez au `.LST`,
faut refaire tout le cycle, sinon les trois fichiers racontent des choses
différentes et vous allez vous arracher les cheveux.

Attention : GEMDOS, donc **noms de fichiers en 8.3**, et le `.LST` doit rester en
**ASCII pur** — pas d'accents, l'éditeur ST n'aime pas ça du tout.

## C'est fini ?

Non. Il reste :

- Un compte à rebours du dévoilement, pour que la salle sache combien il reste
- Une vraie pause, parce qu'en soirée il faut pouvoir arrêter net
- Les scores qui survivent à l'extinction de la machine
- Un écran récapitulatif à la fin, pour repasser sur les réponses
- Et finir le pixel art de l'écran d'accueil, qui n'est pas tout à fait terminé

## Crédits

Code : **Hylst** (Geoffroy)
Coup de main : **Tomchi**
Musiques : **AD**, **DMA SC**, **JESS**
Fonte : Hylst

---

*Fait avec amour et beaucoup de `BMOVE` sur une machine de 1985.*
