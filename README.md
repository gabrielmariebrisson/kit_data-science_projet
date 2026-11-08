# Recommandation de quartiers à Paris

Ce projet construit un système de recommandation de quartiers parisiens à partir de données géographiques, immobilières, environnementales, démographiques et de mobilité.

L'objectif de ce projet est de manipuler des données brutes, de les nettoyer, gérer les valeurs manquantes pour pouvoir utiliser ensuite les données propres sur des algorithmes de ML non supervisé pour construire différents scores thématiques.
Ainsi on pourra proposer la zone de Paris qui correspond mieux au attente d'une famille, d'un étudiant, un jeune activiste etc. tout en respectant ses contraintes de budget et de proximité.

## Démarche suivie

La démarche repose sur trois étapes :

1.Nettoyer et Harmoniser les données brutes ;
2.Agréger les informations à l'échelle des IRIS parisiens et créer les variables utiles ;
3.Calculer les scores puis recommander les IRIS les plus proche du profil recherché.

## Données utilisées

Le projet combine plusieurs familles de données.

### Référentiel géographique

- Contours géographiques des IRIS (unité géographique qui permet de découper Paris en 992 mailles)
- codes et noms des IRIS
- arrondissements et quartiers administratifs
- type d'IRIS : habitat, activité ou divers

### Immobilier

- demandes de valeurs foncières pour les ventes immobilières
- prix de vente et prix au mètre carré
- nombre de pièces
- surface des logements
- loyers de référence par quartier
- caractéristiques du parc de logements issues de l'INSEE

### Transports et mobilité

Les données GTFS d'Île-de-France Mobilités et des stations Vélib sont utilisées pour calculer :

- la distance au métro le plus proche ;
- la distance au train ou RER le plus proche ;
- la distance au tramway le plus proche ;
- la distance au bus et à la station de vélo le plus proche ;
- le nombre de passages de transports en commun sur une journée de référence ;
- le nombre de lignes desservant et nombre de stations de vélo pour chaque IRIS ;
- la part des quais accessibles aux personnes à mobilité réduite.

### Commerces, services et santé

Les données SIRENE sont utilisées pour identifier et compter les établissements ouverts selon leur code NAF.
Le projet calcule notamment la distance aux différents type de commerce et de services.

### Éducation

Les données sur les établissements scolaires permettent de produire :

- la distance à l'école, au collège et au lycée les plus proches ;
- le nombre d'établissements scolaires publics et privés;
- l'indice de position sociale, ou IPS, de l'école et du lycée les plus proches ;
- le taux de réussite et la valeur ajoutée du collège le plus proche.

### Environnement et cadre de vie

Le projet utilise des données sur les espaces verts, les arbres, la qualité de l'air, le bruit.

## Données démographiques et sociales

Les données INSEE sur le logement et l'activité des résidents servent à calculer :

- la part de propriétaires, locataires et logements sociaux ;
- la part de maisons ;
- la part de logements vacants ;
- la taille des logements ;
- le taux d'activité et le taux de chômage ;
- la part d'étudiants, de retraités et des différentes catégories socioprofessionnelles ;
- les modes de déplacement domicile-travail etc.


```text
Données brutes
    |
    v
1. Nettoyage et contrôle qualité
    |
    v
data_cleans/
    |
    v
2. Jointures spatiales et création des variables
    |
    v
save/iris_features.parquet
    |
    v
3. Scoring, typologie et recommandation
    |
    v
save/iris_scores.parquet

```

## 1. Nettoyage des données

Le notebook de nettoyage charge les fichiers depuis le dossier `data/`, puis enregistre les versions préparées dans `data_cleans/`.

### Principales règles de nettoyage

Après une analyse statistique des valeurs aberrantes, manquantes et des doublons, on effectue les nettoyages suivants:

- filtrage sur Paris ;
- sélection des colonnes utiles ;
- lecture par morceaux des fichiers volumineux ;
- suppression des doublons ;
- suppression des géométries ou coordonnées absentes ;
- harmonisation des catégories textuelles ;
- sélection de la période la plus récente ;
- conservation des établissements SIRENE actifs ;
- conservation des quais GTFS, sans dupliquer les gares parentes ;
- exclusion des valeurs immobilières incohérentes ;
- imputation raisonnée des données sur les arbres à partir de la médiane du genre, puis de la médiane globale si nécessaire ;
- filtre géographique final fondé sur le contour exact de Paris.

Les formats de sortie sont :

- GeoJSON pour les données géographiques ;
- Parquet pour les tables classiques.

## 2. Jointure et feature engineering

Le notebook de jointure part des données nettoyées et utilise l'IRIS comme maille commune.

Deux types d'opérations sont principalement utilisés :

### Jointures spatiales

Une observation est rattachée à l'IRIS dans lequel elle se trouve.

### Recherche du plus proche voisin géographique

Pour chaque centre représentatif d'IRIS, le projet calcule la distance à l'équipement le plus proche avec les fonctions de jointure spatiale de GeoPandas.

### Agrégations

Les observations sont agrégées par IRIS afin de produire :

- des comptages ;
- des densités au km² ;
- des sommes ;
- des médianes ;
- des proportions ;
- des distances minimales ;
- des indicateurs pondérés par la surface.

Cette étape produit `iris_features.parquet`, un jeu de données géographique contenant environ 184 variables pour 992 IRIS.

## 3. Construction des scores

Les variables détaillées sont regroupées en critères interprétables :

- transports ;
- accessibilité PMR ;
- écoles ;
- verdure ;
- calme ;
- commerces ;
- santé ;
- sorties ;
- services publics ;
- sport et culture ;
- voiture ;
- grands logements ;
- quartier jeune.

### Transformation en rang percentile

Les variables ont des unités différentes. Une distance est exprimée en mètres, une densité en nombre d'équipements par km² et une proportion en pourcentage.

Pour les rendre comparables, chaque variable est transformée en rang percentile entre 0 et 1 :

- pour une variable positive, une valeur élevée reçoit un meilleur rang ;
- pour une variable négative, comme une distance ou le bruit, une valeur faible reçoit un meilleur rang.

Le score d'un critère correspond ensuite à la moyenne des rangs de ses variables.

Cette approche présente plusieurs avantages :

- elle évite qu'une unité domine les autres ;
- elle réduit l'influence des valeurs extrêmes ;
- elle produit des scores lisibles entre 0 et 1 ;
- elle permet de comparer les IRIS relativement au reste de Paris.

Elle implique cependant que les scores sont relatifs au territoire étudié et ne représentent pas une mesure absolue de qualité.

## 4. Typologie des quartiers avec K-Means

Une classification K-Means est appliquée aux scores afin d'explorer les grands types de quartiers.

Plusieurs nombres de groupes sont comparés avec le score de silhouette. Les scores obtenus restent faibles, ce qui montre que les quartiers parisiens ne forment pas des catégories parfaitement séparées. La typologie est donc utilisée comme outil descriptif, et non comme vérité géographique stricte.

Six groupes sont finalement conservés et nommés à partir de leurs critères les plus caractéristiques, par exemple :

- sorties et commerces ;
- quartier jeune et santé ;
- calme et services publics ;
- grands logements et quartier jeune ;
- verdure et quartier jeune ;
- sorties et accessibilité PMR.

## 5. Moteur de recommandation k-NN

Le moteur de recommandation utilise l'algorithme des k plus proches voisins, ou k-NN.

### Représentation du quartier idéal

Pour chaque critère choisi, le quartier idéal possède un score égal à 1.

Les préférences de l'utilisateur sont représentées par des poids de 0 à 3 :

- 0 : critère ignoré ;
- 1 : critère peu important ;
- 2 : critère important ;
- 3 : critère prioritaire.

Un critère noté 3 influence ainsi trois fois plus la distance qu'un critère noté 1.

### Filtrage avant recommandation

Avant l'exécution du k-NN, les candidats peuvent être filtrés selon :

- le prix maximal au mètre carré ;
- le loyer maximal au mètre carré ;
- une distance maximale à certains équipements ;
- le type résidentiel de l'IRIS.

Le filtrage en amont évite de recommander un quartier très bien noté mais incompatible avec les contraintes essentielles de l'utilisateur.

### Profils actuellement définis

- **Famille** : écoles, verdure, calme, grands logements, commerces et santé ;
- **Jeune actif** : transports, sorties, sport, culture et commerces ;
- **Étudiant** : transports, caractère jeune, sorties, sport, culture et services publics ;
- **Retraité** : calme, santé, commerces et verdure ;
- **Automobiliste** : stationnement, voiture, calme et commerces.

Le modèle renvoie par défaut les dix IRIS les plus proches du profil idéal.

## Validation

La validation reste partielle, car le projet ne dispose pas d'un jeu de vérité terrain contenant des notes de satisfaction par quartier et par profil.

Pour aller plus loin, il serait utile de collecter des évaluations utilisateurs puis de mesurer des indicateurs comme Precision@K, Recall@K, NDCG@K ou le taux de clic et de sélection.






