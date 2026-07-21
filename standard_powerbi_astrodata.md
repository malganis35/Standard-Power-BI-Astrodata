# Standard Power BI — Astrodata

## Objectif

Les modèles Power BI doivent être lisibles, maintenables et compréhensibles par un utilisateur métier et technique.

> **Langue du modèle** : tous les noms de tables, colonnes et mesures sont rédigés en **anglais**. Les descriptions sont également en anglais. La documentation interne peut rester en français.

---

## 1. Architecture & Organisation du projet

- **Principe du modèle unique** : Séparer strictement le jeu de données (*Semantic Model*) de la couche de restitution (*Report*). Un Semantic Model peut alimenter plusieurs rapports.
- **Format de fichier** : Travailler au format **Power BI Project (.pbip)** pour permettre le suivi des modifications via Git (syntaxe TMDL). Éviter le format `.pbix` pour les projets collaboratifs.

---

## 2. Power Query & Ingestion (M)

- **Query Folding (Pliage de requête)** : Privilégier le pliage vers la source autant que possible, en particulier pour les filtres et sélections de colonnes. Vérifier le pliage via le clic droit sur une étape → *View Native Query*. Les opérations complexes (colonnes conditionnelles, jointures custom) peuvent casser le pliage — c'est acceptable si les étapes amont restent pliées.
- **Typage explicite** : Définir le type de chaque colonne dans Power Query. Ne pas laisser Power BI inférer les types dans le modèle.
- **Nettoyage précoce** : Supprimer les colonnes inutiles (`Remove Columns`) dès les premières étapes de la requête, avant toute transformation coûteuse.
- **Tables de staging** : Les tables intermédiaires (staging, requêtes de référence) doivent avoir le chargement désactivé (`Enable Load` → off) pour ne pas apparaître dans le modèle.

---

## 3. Nommage des tables

### Préfixes obligatoires

| Type | Préfixe | Exemple |
|---|---|---|
| Table de faits | `FACT_` | `FACT_SALES` |
| Table de dimensions | `DIM_` | `DIM_CUSTOMER` |
| Table de paramètres (Field Parameters) | `PARAM_` | `PARAM_MEASURE_SELECTOR` |
| Table de liaison (many-to-many) | `BRIDGE_` | `BRIDGE_ORDER_PRODUCT` |

Toutes les mesures doivent être regroupées dans une table dédiée appelée `_MEASURES`.

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

## 4. Description des tables et colonnes

Toutes les descriptions sont rédigées en **anglais**.

### Tables

Chaque table doit avoir une description précisant :
- son rôle dans le modèle ;
- son type (fact table, dimension, bridge, etc.) ;
- son usage principal.

**Exemple :**
> `FACT_SALES` — Fact table storing all sales transactions at order line level. Used to calculate revenue, quantity sold, and margin KPIs.

### Colonnes

Toutes les colonnes doivent avoir une description métier. En particulier, les colonnes dont le rôle n'est pas évident doivent faire l'objet d'une documentation précise :
- les clés techniques ;
- les colonnes de statut ou de code ;
- les colonnes avec des valeurs encodées.

---

## 5. Modélisation

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

## 6. Mesures DAX

### Table de mesures

- Regrouper toutes les mesures dans la table `_MEASURES`, créée via `Enter Data` avec une table vide.
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

### Variables DAX (VAR / RETURN)

Utiliser `VAR` / `RETURN` pour toute mesure avec plus d'un calcul intermédiaire. Cela améliore la lisibilité, évite de répéter des sous-expressions, et facilite le débogage.

```DAX
Gross Margin % =
VAR _GrossMargin = [Gross Margin]
VAR _TotalRevenue = [Total Revenue]
RETURN
    DIVIDE(_GrossMargin, _TotalRevenue, 0)
```

Éviter l'imbrication profonde de fonctions sans variables :

```DAX
-- ❌ difficile à lire et à déboguer
Gross Margin % = DIVIDE(SUM(FACT_SALES[Margin]), SUM(FACT_SALES[Revenue]), 0)

-- ✅ préférer
Gross Margin % =
VAR _Margin = SUM(FACT_SALES[Margin])
VAR _Revenue = SUM(FACT_SALES[Revenue])
RETURN DIVIDE(_Margin, _Revenue, 0)
```

### Documentation DAX

Commenter les mesures complexes directement dans le code DAX :

```DAX
-- Gross Margin % = Gross Margin / Total Revenue
-- Excludes returns (OrderStatus <> "Returned")
Gross Margin % =
VAR _Margin = [Gross Margin]
VAR _Revenue = [Total Revenue]
RETURN
    DIVIDE(_Margin, _Revenue, 0)
```

### Sélecteur dynamique (DATATABLE + SWITCH)

Pour créer un sélecteur de métriques dynamique via Field Parameter :

```DAX
-- Table de sélection statique
Metric Selector =
DATATABLE(
    "Metric Name", STRING,
    {
        {"Revenue"},
        {"Gross Margin"},
        {"Units Sold"}
    }
)

-- Mesure de commutation
Selected Metric =
SWITCH(
    SELECTEDVALUE('Metric Selector'[Metric Name]),
    "Revenue",      [Total Revenue],
    "Gross Margin", [Gross Margin],
    "Units Sold",   [Total Units Sold],
    BLANK()
)
```

---

## 7. Calendrier DAX

### Génération

Cibler dynamiquement la plage de dates à partir des tables de faits plutôt que d'utiliser `CALENDARAUTO()`, qui peut générer des plages disproportionnées en présence de valeurs aberrantes ou de colonnes de type date non prévues dans le modèle.

```DAX
DIM_DATE =
VAR _MinYear = YEAR(MIN(FACT_SALES[OrderDate]))
VAR _MaxYear = YEAR(MAX(FACT_SALES[OrderDate]))
VAR _StartDate = DATE(_MinYear, 1, 1)
VAR _EndDate = DATE(_MaxYear, 12, 31)
RETURN
    ADDCOLUMNS(
        CALENDAR(_StartDate, _EndDate),
        "Year",            YEAR([Date]),
        "Quarter",         "Q" & QUARTER([Date]),
        "QuarterNumber",   QUARTER([Date]),
        "Month",           FORMAT([Date], "MMMM"),
        "MonthNumber",     MONTH([Date]),
        "MonthShort",      FORMAT([Date], "MMM"),
        "Week",            WEEKNUM([Date], 2),
        "DayOfWeek",       FORMAT([Date], "dddd"),
        "DayOfWeekNumber", WEEKDAY([Date], 2),
        "IsWeekend",       WEEKDAY([Date], 2) >= 6,
        "YearMonth",       FORMAT([Date], "YYYY-MM"),
        "YearQuarter",     YEAR([Date]) & " Q" & QUARTER([Date])
    )
```

### Configuration obligatoire

- **Marquer la table comme table de dates** (`Mark as date table`) en sélectionnant la colonne `Date`.
- Masquer la colonne de date brute dans les tables de faits après avoir créé la relation avec `DIM_DATE`.
- La colonne `MonthNumber` doit être utilisée pour **trier** la colonne `Month` (`Sort by Column`).

---

## 8. Design des rapports

### Avant de commencer : le Dashboard Canvas

Avant de créer un rapport, définir les besoins avec un canvas :
- **Audience** : qui sont les utilisateurs ? quel est leur niveau de lecture des données ?
- **Objectif** : quelle question le rapport doit-il répondre ? quelles décisions doit-il supporter ?
- **KPIs** : quelles métriques mesurer ? quelle est la définition de chaque KPI ?
- **Dimensions** : quels axes d'analyse (temps, géographie, produit...) ?
- **Granularité** : quel est le niveau de détail le plus fin nécessaire ?
- **Partage** : comment le rapport sera-t-il distribué (Service, app, export PDF) ?
- **Fréquence de rafraîchissement** : temps réel, quotidien, hebdomadaire ?

### Structure des pages

- Commencer par une **page de couverture** avec le titre, la date de mise à jour et le contexte.
- Prévoir une **page de synthèse** avec les KPIs principaux en haut.
- Ne pas dépasser **4 visuels par page** pour éviter la surcharge cognitive.
- Prévoir une navigation claire entre les pages (boutons, signets).

### Layout

- Adopter le **patron Z ou F** pour positionner les éléments : l'œil du lecteur suit naturellement ces trajectoires.
- Placer les **KPIs en haut**, les filtres dans un panneau latéral dédié.
- Utiliser les **espaces blancs** — une page surchargée est moins lisible qu'une page aérée.
- Donner un **titre clair** à chaque visuel, idéalement sous forme de question (*"Which region drives the most revenue?"*).

### Choix des visuels

| Besoin | Type de visuel recommandé |
|---|---|
| Comparer des catégories | Barres / Histogrammes |
| Montrer une tendance dans le temps | Courbe |
| Montrer des proportions | Graphique en secteurs / Donut (max 5 segments) |
| Identifier une corrélation | Nuage de points |
| Afficher une valeur clé | Card / KPI |
| Détail tabulaire | Table / Matrice |

Ressource : [Visual Vocabulary](https://gramener.github.io/visual-vocabulary-vega/)

### Thème JSON

- Tout rapport doit utiliser un **fichier JSON de thème** pour standardiser couleurs, polices et styles.
- Ne pas configurer les couleurs manuellement sur chaque visuel.
- Ressources : [bibb.pro/apps/theme-generator](https://bibb.pro/apps/theme-generator/), [themes.powerbi.tips](https://themes.powerbi.tips/themes/gallery)

### Fonds et templates

- Créer les fonds de rapport dans **PowerPoint ou Figma**, pas directement dans Power BI (lent et peu flexible).
- Exporter les fonds en **SVG** pour une résolution optimale sur tous les écrans.
- Importer le fond via `Canvas Background` dans les paramètres de la page.

### Navigation & interactions

- Configurer les **interactions visuelles** explicitement (filtre, surlignage, ou aucune interaction) pour éviter les comportements inattendus.
- Utiliser les **Drill-Down** pour naviguer dans une hiérarchie au sein d'un même visuel (ex. : Année → Trimestre → Mois).
- Utiliser les **Drill-Through** pour naviguer vers une page de détail (ex. : cliquer sur un pays → page de détail par ville).
- Utiliser les **Signets (Bookmarks)** et **Boutons** pour créer des vues alternatives ou des réinitialisations de filtres.

---

## 9. Sécurité (RLS) (optionnel)

- Définir les rôles RLS (*Row-Level Security*) dans Power BI Desktop avant la publication.
- Nommer les rôles de manière explicite et orientée métier : `Region_EMEA`, `Sales_Manager`, `Finance_ReadOnly`.
- Tester chaque rôle avec la fonctionnalité `View as` avant publication.
- Documenter dans ce standard les rôles existants et leur périmètre de données.

---

## 10. Performance

- Préférer le mode **Import** au mode DirectQuery sauf besoin de fraîcheur temps réel justifié.
- Limiter le nombre de colonnes importées au strict nécessaire (supprimer les colonnes inutilisées à la source dans Power Query).
- Éviter les **colonnes calculées à haute cardinalité** (texte libre, identifiants uniques) — elles augmentent la taille du modèle.
- Désactiver le chargement des tables intermédiaires Power Query qui ne doivent pas apparaître dans le modèle (`Enable Load` → off).

---

## 11. Power BI Service

### Organisation des workspaces

- Créer un workspace par **domaine ou projet**, pas un workspace unique pour tous les rapports.
- Nommer les workspaces de façon explicite : `[Domaine] - [Environnement]` → `Sales Analytics - Prod`.
- Distinguer les environnements : **Dev**, **UAT**, **Prod**.

### Publication

- Publier depuis Power BI Desktop via `Publish` → sélectionner le workspace cible.
- Vérifier après publication que le rapport et le Semantic Model sont bien présents dans le workspace.
- Ne jamais modifier un rapport directement dans le Service — toujours republier depuis Desktop.

### Rafraîchissement des données

- Configurer un **rafraîchissement planifié** (*Scheduled Refresh*) dans les paramètres du Semantic Model.
- Si la source est on-premise (ex. : Snowflake derrière un réseau interne), configurer la **Data Gateway**.
- Documenter la fréquence de rafraîchissement dans la description du Semantic Model.

### Partage et permissions

- Gérer les accès au niveau du **workspace** (rôles : Admin, Member, Contributor, Viewer).
- Utiliser les **Power BI Apps** pour distribuer des rapports à une audience large en lecture seule.
- Ne pas partager les rapports en lien direct avec des droits d'édition sauf besoin explicite.

---

## 12. Checklist avant publication

**Obligatoire :**
- [ ] Toutes les tables ont un préfixe et une description.
- [ ] Toutes les colonnes ont une description métier.
- [ ] Toutes les colonnes techniques sont masquées.
- [ ] Les agrégations automatiques des colonnes numériques sont vérifiées.
- [ ] Toutes les mesures ont un format défini et sont organisées en Display Folders.
- [ ] Les mesures complexes sont commentées et utilisent VAR/RETURN.
- [ ] Les relations sont en cardinalité many-to-one, direction simple.
- [ ] `DIM_DATE` est marquée comme table de dates.
- [ ] Aucune table intermédiaire Power Query n'est chargée dans le modèle.
- [ ] Le rapport utilise un thème JSON.
- [ ] Le rapport a une page de couverture et une page de synthèse KPI.

**Optionnel :**
- [ ] Les rôles RLS sont définis et testés.
- [ ] Un rafraîchissement planifié est configuré dans le Service.