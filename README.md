# Dessiner le Tour de France : clustering et optimisation de tournées

> Comment répartir 120 villages français en 21 étapes cohérentes, puis trouver dans chacune
> l'ordre de passage le plus court ?

De 46 789 km à 4 063 km. Une division par 11,5, sur un problème dont l'énumération
exhaustive est physiquement impossible.

## Le problème

Un comité a sélectionné 120 villages. Il faut les répartir en 21 étapes géographiquement
cohérentes, puis ordonner chaque étape pour minimiser les kilomètres un départ, une arrivée,
pas de retour au point de départ.

C'est un problème du voyageur de commerce sur chemin ouvert. Pour 120 villages, le nombre
d'ordres distincts s'écrit avec 199 chiffres. À titre de comparaison, l'univers observable
compte environ 10^80 atomes. La force brute est hors de question.

## L'approche

Diviser pour régner, à deux niveaux.

| Étape | Méthode | Résultat |
|---|---|---|
| Découper la France | K-Means / CAH sur coordonnées projetées en km | 21 étapes |
| Ordonner chaque étape | Plus proche voisin (glouton) | Parcours initial |
| Polir | 2-opt (décroisement de segments) | Optimum local |
| Choisir | Tous les départs testés par étape | Meilleur parcours |

Le découpage transforme un problème à 199 chiffres de possibilités en 21 sous-problèmes de
2 à 12 villages, chacun résolu quasi optimalement.

Détail technique : les coordonnées GPS sont projetées en kilomètres avant clustering
(un degré de longitude vaut 78 km à la latitude moyenne des villages, contre 111 km pour un
degré de latitude). Les distances sont calculées par la formule de haversine, exacte sur une
sphère.

## Le point de méthode : quand la métrique se trompe

Deux algorithmes de clustering ont été comparés sur deux critères qui se contredisent.

| | Silhouette | Taille des étapes |
|---|---:|---|
| K-Means | 0,461 | 2 à 12 villages |
| **CAH** | **0,467** ✓ | **1 à 14 villages** ✗ |

La CAH gagne sur la métrique — de six millièmes. Mais elle produit une étape à un seul
village, ce qui n'a aucun sens sportif.

Le K-Means a été retenu, moins bon sur le papier, utilisable dans la réalité.

Une métrique optimisée n'est pas un objectif atteint. La silhouette mesure la netteté
géométrique des groupes, pas leur utilité pour organiser une course cycliste.

## Le résultat

| | |
|---|---:|
| Ordre alphabétique du fichier | 46 789 km |
| **Tracé optimisé** | **4 063 km** |
| Étape reine (n° 11) | 362 km |
| Étape la plus courte (n° 12) | 19 km |
| Moyenne par étape | 193 km |

Pour référence, un vrai Tour de France fait environ 3 500 km sur 21 étapes, soit ~150 km
par étape.

## Analyse critique : d'où viennent les étapes trop longues ?

Neuf étapes sur vingt et une dépassent 200 km, contre une moyenne réelle de 150 km. Deux
hypothèses, testées plutôt que supposées.

### Hypothèse 1 : Le découpage est déséquilibré (réfutée)

Le K-Means ne contraint pas la taille des groupes. J'ai implémenté un K-Means sous contrainte
de capacité (6 villages maximum par étape) pour voir si l'équilibre corrigeait le problème.

| Découpage | Total | Étape max | > 200 km | Tailles | Silhouette |
|---|---:|---:|---:|---|---:|
| K-Means (retenu) | 4 063 km | 362 km | 9/21 | 2–12 | 0,461 |
| CAH | 3 930 km | 324 km | 9/21 | 1–14 | 0,467 |
| K-Means équilibré | **4 820 km** | **454 km** | **12/21** | 3–6 | 0,339 |

L'équilibrage aggrave tout : +19 % de kilomètres, étape reine plus longue, trois étapes
excessives de plus. Forcer six villages par groupe oblige à rattacher des villages éloignés
à des groupes déjà pleins ailleurs : on échange de la compacité contre de l'équilibre, et
c'est la compacité qui compte ici.

Un résultat négatif, mais qui valait d'être mesuré sans le test, « il suffirait d'équilibrer »
serait resté une intuition plausible et fausse.

### Hypothèse 2 : La liste de villages est trop dispersée (confirmée)

| Mesure | Valeur |
|---|---:|
| Distance médiane au plus proche voisin | 26 km |
| Distance maximale (Locronan, Bretagne) | 120 km |
| Villages à plus de 50 km de tout voisin | **29 / 120** |
| Part du tracé portée par les 10 liaisons les plus longues | **24 %** |

Près d'un quart des villages sont des isolés qu'aucun algorithme ne peut rattacher sans
allonger le trajet. Et l'effet est concentré : dix liaisons sur quatre-vingt-dix-neuf portent
le quart du kilométrage.

### Combien d'étapes faudrait-il ?

| Étapes | Total | Moyenne |
|---:|---:|---:|
| 21 | 4 063 km | 193 km |
| **25** | 3 684 km | **147 km** ✓ |
| 30 | 3 206 km | 107 km |

Il faudrait 25 étapes pour tenir la moyenne d'un vrai Tour. Le format en impose 21.

Conclusion : les 120 villages proposés sont trop dispersés pour tenir en 21 étapes de
gabarit réaliste. Ce n'est pas un problème d'algorithme, c'est un problème de cahier des charges.

## Recommandation

1. Densifier les zones creuses (Bretagne, Massif central) en ajoutant des villages
   intermédiaires.
2. Retirer les isolés les plus coûteux, ou les traiter comme villes-départ après transfert.
3. Prévoir une arrivée à Paris absente de la liste actuelle.

## Limites

Les distances sont à vol d'oiseau : le kilométrage routier réel sera sensiblement supérieur.
Le relief n'est pas modélisé, alors qu'il structure un vrai Tour. Le 2-opt garantit un optimum
local, pas global sur des étapes de cette taille l'écart est probablement faible, mais
non nul. Les transferts entre l'arrivée d'une étape et le départ de la suivante ne sont pas
comptés.

## Ce que le projet m'a appris

Le découpage vaut mieux que la force brute. Un problème impossible devient trivial une fois
partitionné c'est vrai bien au-delà du voyageur de commerce.

Une métrique ne connaît pas la question métier. Le choix du K-Means contre la CAH s'est
joué sur une contrainte que la silhouette ignorait complètement.

**Un résultat négatif est un résultat.** Le K-Means équilibré ne marche pas, et c'est
précisément ce test qui permet d'affirmer que le problème vient des données et non du code.

## Reproduire

```bash
git clone https://github.com/badaouihakimou/tour-de-france-clustering.git
cd tour-de-france-clustering
pip install -r requirements.txt
jupyter notebook projet_03.ipynb
```

Ou en un clic via le badge Colab en haut de ce fichier — pense à téléverser `utils.py` et
`villages_2027.csv` quand la première cellule le demande.

## Structure

```txt
├── projet_03.ipynb        # le notebook complet
├── utils.py               # chargement, cartes, livre de route
├── villages_2027.csv      # les 120 villages
├── requirements.txt
└── README.md
```

## Notions couvertes

Clustering (K-Means, classification ascendante hiérarchique, linkage de Ward), évaluation non
supervisée (score de silhouette), projection de coordonnées géographiques, formule de haversine,
problème du voyageur de commerce, heuristiques gloutonnes, recherche locale 2-opt, explosion
combinatoire.

Exercice issu du [Cahier de Vacances Data](https://machinelearnia.com/) de Machine Learnia
(Guillaume Saint-Cirgue). Le code, l'analyse critique et la Partie 5 sont les miens.
