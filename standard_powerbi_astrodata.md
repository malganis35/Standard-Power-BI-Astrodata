# Standard Power BI Astrodata

## Objectif

Les modèles Power BI doivent être lisibles, maintenables et compréhensibles par un utilisateur métier et technique.

## Nommage des tables

- Les tables de faits doivent être au pluriel.
- Les dimensions doivent être au singulier.
- Le nom des tables doit être en majuscule.
- Le nom des colonnes doit être en CamelCase (Majuscule pour la première lettre en ensuite minuscule)
- Tous les noms de tables doivent être préfixées: `FACT_` pour les tables de faits, `DIM_` pour les tables de dimensions ou `PARAM_` pour les paramètres de champ.
- Les noms doivent être courts, clairs et orientés métier.

## Description des tables

Toutes les descriptions doivent être rédigées en anglais.

Chaque table doit avoir une courte description.

La description doit préciser :

- le rôle de la table ;
- si c’est une table de fait ou une dimension ;
- son usage principal dans le modèle.

## Modélisation

- Favoriser un modèle en étoile.
- Éviter les relations ambiguës.
- Masquer les colonnes techniques inutiles pour les utilisateurs.
- Vérifier les colonnes numériques pour éviter les agrégations automatiques non pertinentes.

## Calendrier DAX

Si un calendrier DAX est utilisé, privilégier `CALENDARAUTO()` pour générer automatiquement la plage de dates du modèle.

Exemple :

```DAX
DimDate =
CALENDARAUTO()