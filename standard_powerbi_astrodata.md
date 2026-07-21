# Standard Power BI — Astrodata

## Objectif

Les modèles Power BI doivent être lisibles, maintenables et compréhensibles par un utilisateur métier et technique.

> **Langue du modèle** : tous les noms de tables, colonnes et mesures sont rédigés en **anglais**. Les descriptions sont également en anglais. La documentation interne peut rester en français.

---

## 1. Nommage des tables

### Préfixes obligatoires

| Type | Préfixe | Exemple |
|---|---|---|
| Table de faits | `FACT_` | `FACT_SALES` |
| Table de dimensions | `DIM_` | `DIM_CUSTOMER` |
| Table de paramètres (Field Parameters) | `PARAM_` | `PARAM_MEASURE_SELECTOR` |
| Table de liaison (many-to-many) | `BRIDGE_` | `BRIDGE_ORDER_PRODUCT` |

Toutes les mesures doivent être dans une table appelée `_MEASURES`

### Règles de nommage

- Les noms de tables sont en **MAJUSCULES** avec underscore comme séparateur : `FACT_SALES_ORDERS`.
- Les tables de faits sont au **pluriel** : `FACT_SALES`, `FACT_TRANSACTIONS`.
- Les tables de dimensions sont au **singulier** : `DIM_CUSTOMER`, `DIM_PRODUCT`.
- Les noms doivent être **courts, clairs et orientés métier**.

### Colonnes

- Les noms de colonnes utilisent le **PascalCase** (première lettre de chaque mot en majuscule) : `OrderDate`, `CustomerName`, `TotalAmount`.
- Les clés primaires utilisent le suffixe `Key` : `CustomerKey`, `ProductKey`, `DateKey`.
- Les clés étrangères utilisent le même nom que la clé primaire référencée.
- Éviter les abréviations non évidentes : préférer `OrderDate` à `OrdDt`.

**Exemples valides :**
```
CustomerKey   ✅
Cust_Key      ❌
customerkey   ❌
```

---

## 2. Description des tables et colonnes

Toutes les descriptions sont rédigées en **anglais**.

### Tables

Chaque table doit avoir une description précisant :
- son rôle dans le modèle ;
- son type (fact table, dimension, bridge, etc.) ;
- son usage principal.

**Exemple :**
> `FACT_SALES` — Fact table storing all sales transactions at order line level. Used to calculate revenue, quantity sold, and margin KPIs.

### Colonnes

Toutes les colonnes doivent avoir une description métier. En particulier, les colonnes dont le rôle n'est pas évident doivent faire l'objet d'une documentation précise comme par exemple :
- les clés techniques ;
- les colonnes de statut ou de code ;
- les colonnes avec des valeurs encodées.

---

## 3. Modélisation

### Schéma en étoile

- Favoriser un **modèle en étoile** : une table de faits centrale reliée à des dimensions.
- Éviter les **modèles en flocon** (snowflake) sauf si une dimension est trop volumineuse pour être dénormalisée.
- Les tables **BRIDGE_** servent à résoudre les relations many-to-many — ne pas créer de relation directe many-to-many entre une fact et une dimension.

### Relations

- Cardinalité préférée : **many-to-one** (de la table de faits vers la dimension).
- Direction de filtrage : **simple (single)** par défaut. La bidirectionnelle (`Both`) doit être justifiée et documentée — elle peut causer des ambiguïtés et des problèmes de performance.
- Préférer les relations **actives**. Les relations inactives sont utilisables via `USERELATIONSHIP()` en DAX, mais leur existence doit être documentée dans la description de la table.
- Éviter les relations **ambiguës** (chemins multiples entre deux tables).

### Colonnes et visibilité

- **Masquer** toutes les colonnes techniques inutiles pour les utilisateurs (clés, colonnes de tri internes, colonnes intermédiaires de calcul).
- Les colonnes masquées restent utilisables en DAX — les masquer ne les supprime pas du modèle.
- Vérifier que les **colonnes numériques** non destinées à être agrégées ont leur agrégation par défaut définie sur `Don't summarize` (ex. : `CustomerKey`, `PostalCode`).

### Colonnes calculées vs mesures

| | Colonne calculée | Mesure |
|---|---|---|
| Calculée | Au chargement | À la volée |
| Contexte | Ligne par ligne | Contexte de filtre |
| Impact mémoire | Élevé | Faible |
| Usage recommandé | Segmentation, catégories, clés de relation | Agrégations, KPIs, calculs dynamiques |

> **Règle** : utiliser une **mesure** par défaut. Créer une colonne calculée uniquement si la valeur est nécessaire comme axe de filtre, de regroupement ou de relation.

---

## 4. Mesures DAX

### Table de mesures

- Regrouper toutes les mesures dans une **table dédiée** sans données (créées via `Enter Data` avec une table vide).
- Nommer cette table avec le préfixe `_` : `_MEASURES`.
- Ne pas mélanger mesures et colonnes dans une même table.

### Nommage des mesures

- Utiliser le **PascalCase avec espaces** pour la lisibilité dans les visuels : `Total Revenue`, `Gross Margin %`, `YTD Sales`.
- Être explicite sur le type de calcul : préférer `Total Revenue` à `Revenue`, `YTD Sales` à `Sales YTD`.
- Les mesures en pourcentage se terminent par `%` : `Gross Margin %`.

### Display Folders

Organiser les mesures en dossiers d'affichage (`Display Folders`) par domaine :

```
_MEASURES
├── Revenue
│   ├── Total Revenue
│   ├── YTD Revenue
│   └── Revenue vs. Prior Year
├── Margin
│   ├── Gross Margin
│   └── Gross Margin %
└── Volume
    ├── Total Orders
    └── Total Units Sold
```

### Format des mesures

Toujours définir le format d'affichage d'une mesure :
- Montants : `#,##0.00 €` ou `$ #,##0`
- Pourcentages : `0.00%`
- Entiers : `#,##0`

### Documentation DAX

Commenter les mesures complexes directement dans le code DAX :

```DAX
-- Gross Margin % = Gross Margin / Total Revenue
-- Excludes returns (OrderStatus <> "Returned")
Gross Margin % =
DIVIDE(
    [Gross Margin],
    [Total Revenue],
    0
)
```

---

## 5. Calendrier DAX

### Génération

Utiliser `CALENDARAUTO()` pour générer automatiquement la plage de dates couverte par le modèle.

```DAX
DIM_DATE =
VAR _Calendar = CALENDARAUTO()
VAR _Result =
    ADDCOLUMNS(
        _Calendar,
        "Year",           YEAR([Date]),
        "Quarter",        "Q" & QUARTER([Date]),
        "QuarterNumber",  QUARTER([Date]),
        "Month",          FORMAT([Date], "MMMM"),
        "MonthNumber",    MONTH([Date]),
        "MonthShort",     FORMAT([Date], "MMM"),
        "Week",           WEEKNUM([Date], 2),
        "DayOfWeek",      FORMAT([Date], "dddd"),
        "DayOfWeekNumber",WEEKDAY([Date], 2),
        "IsWeekend",      WEEKDAY([Date], 2) >= 6,
        "YearMonth",      FORMAT([Date], "YYYY-MM"),
        "YearQuarter",    YEAR([Date]) & " Q" & QUARTER([Date])
    )
RETURN _Result
```

### Configuration obligatoire

- **Marquer la table comme table de dates** (`Mark as date table`) en sélectionnant la colonne `Date`.
- Masquer la colonne `Date` de la table de faits après avoir créé la relation avec `DIM_DATE`.
- La colonne `MonthNumber` doit être utilisée pour **trier** la colonne `Month` (Sort by Column).

---

## 6. Sécurité (RLS) (optionnel)

- Définir les rôles RLS (*Row-Level Security*) dans Power BI Desktop avant la publication.
- Nommer les rôles de manière explicite et orientée métier : `Region_EMEA`, `Sales_Manager`, `Finance_ReadOnly`.
- Tester chaque rôle avec la fonctionnalité `View as` avant publication.
- Documenter dans ce standard les rôles existants et leur périmètre de données.

---

## 7. Performance

- Préférer le mode **Import** au mode DirectQuery sauf besoin de fraîcheur temps réel justifié.
- Limiter le nombre de colonnes importées au strict nécessaire (supprimer les colonnes inutilisées à la source dans Power Query).
- Éviter les **colonnes calculées à haute cardinalité** (texte libre, identifiants uniques) — elles augmentent la taille du modèle.
- Désactiver le chargement des tables intermédiaires Power Query qui ne doivent pas apparaître dans le modèle (`Enable load` → off).

---

## 8. Checklist avant publication

Obligatoire:
- [ ] Toutes les tables ont un préfixe et une description.
- [ ] Toutes les colonnes techniques sont masquées.
- [ ] Les agrégations automatiques des colonnes numériques sont vérifiées.
- [ ] Toutes les mesures ont un format défini et sont organisées en Display Folders.
- [ ] Les relations sont en cardinalité many-to-one, direction simple.
- [ ] `DIM_DATE` est marquée comme table de dates.
- [ ] Aucune table intermédiaire Power Query n'est chargée dans le modèle.

Optionnel:
- [ ] Les rôles RLS sont définis et testés.