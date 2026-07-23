# Standard Python — Astrodata

## Objectif

Le code Python doit être **lisible, testable, reproductible et maintenable**, aussi bien par un data scientist en exploration que par un développeur qui industrialise. Ce standard décrit les conventions appliquées à tout projet généré depuis le boilerplate `cookiecutter-astrodata-squeleton`.

> **Langue du code** : tout le code (noms de modules, fonctions, variables, classes), les docstrings et les commentaires sont rédigés en **anglais**. La documentation destinée aux clients et étudiants peut rester en **français**. Les messages de commit suivent la convention **Conventional Commits** en anglais.

> **Principe directeur** : le boilerplate *applique* ces règles mécaniquement (pre-commit, Ruff, mypy, CI). Ce document en explique le *pourquoi*. En cas de doute, la configuration du boilerplate fait foi.

---

## 1. Architecture & Organisation du projet

- **Séparation logique / interfaces** : le cœur métier vit dans `src/<package>/core/` et ne dépend jamais de la couche de restitution. Les interfaces (`api/` FastAPI, `app/` Streamlit) *consomment* le core, jamais l'inverse. Cette séparation permet de déployer l'API seule, de réutiliser le core dans Spark, ou de changer de frontend sans toucher aux pipelines.
- **Structure standard** : ne pas dévier de l'arborescence du boilerplate.

```text
my_project/
├── app/                    # Frontend Streamlit (interface humaine)
├── data/                   # Données (git-ignoré, versionné via DVC si > 10 Mo)
│   ├── raw/                # Source immuable — ne jamais modifier
│   ├── interim/            # Étapes intermédiaires
│   ├── processed/          # Données prêtes pour le modèle
│   └── external/           # Sources tierces
├── docs/                   # Documentation (Sphinx/RST)
├── logs/                   # Sorties de logs (git-ignoré)
├── models/                 # Modèles sérialisés
├── notebooks/              # Exploration uniquement
├── scripts/                # Scripts one-shot / batch
├── src/
│   └── <package>/
│       ├── api/            # Backend FastAPI
│       ├── core/           # Logique métier réutilisable
│       │   ├── data_io/    # Extraction, chargement, nettoyage initial
│       │   ├── features/   # Feature engineering, transformations
│       │   ├── models/     # Définition, entraînement, inférence
│       │   ├── utils/      # Utilitaires partagés (paths, version, logging)
│       │   └── visualization/  # Fonctions de plot réutilisables
│       └── front/          # Composants front partagés
├── tests/
│   ├── unit_test/          # Tests unitaires (isolés, rapides)
│   ├── integration/        # Tests d'intégration (composants combinés)
│   └── functional/         # Tests bout-en-bout
├── pyproject.toml          # Source unique de configuration
└── VERSION                 # Version courante (synchronisée par commitizen)
```

- **Layout `src/`** : le code packagé vit sous `src/<package>/`. Cela force à installer le package (`uv sync`) pour l'importer, ce qui évite les faux positifs d'import et garantit que les tests s'exécutent contre le package installé, pas contre le répertoire de travail.
- **Interdiction des chemins relatifs en dur** : ne jamais écrire `"../../data"`. Importer les constantes depuis `core.utils.paths` pour que le code tourne à l'identique en notebook, en terminal ou en conteneur.

```python
from my_project.core.utils.paths import RAW_DATA_DIR   # ✅
df = pd.read_csv("../../data/raw/sales.csv")            # ❌
```

---

## 2. Gestion de l'environnement & dépendances

- **`uv` est la source unique de vérité** : ne jamais utiliser `pip` ou `conda` manuellement. Ajouter une dépendance via `uv add <package>` (met à jour `pyproject.toml` automatiquement), synchroniser via `uv sync`.
- **Committer `uv.lock`** : garantit que chaque développeur et la CI utilisent des versions strictement identiques. Le lock est mis à jour automatiquement au bump de version (`pre_bump_hooks`).
- **Séparer prod et dev** : les dépendances runtime vont dans `[project.dependencies]`, l'outillage (pytest, ruff, mypy, pre-commit…) dans le groupe `dev`. L'image Docker de prod installe avec `--no-dev`.
- **`requires-python`** : figer la version Python minimale dans `pyproject.toml`. La CI et le Dockerfile s'alignent sur cette valeur.

**Commandes de référence :**

| Action | Commande |
|---|---|
| Installer tout (prod + dev) | `make install` (`uv sync`) |
| Installer + hooks pre-commit | `make dev-install` |
| Ajouter une dépendance runtime | `uv add pandas` |
| Ajouter une dépendance dev | `uv add --dev pytest-xdist` |

---

## 3. Nommage

### Modules, packages, fichiers

- **`snake_case`**, courts et orientés domaine : `data_io`, `feature_store`, `paths.py`.
- Un module = une responsabilité claire. Éviter les `utils.py` fourre-tout au-delà du core.

### Objets Python

| Élément | Convention | Exemple |
|---|---|---|
| Variable / fonction | `snake_case` | `raw_data_dir`, `get_project_version()` |
| Constante (module-level) | `UPPER_SNAKE_CASE` | `RAW_DATA_DIR`, `MODELS_DIR` |
| Classe | `PascalCase` | `FeaturePipeline`, `ChurnModel` |
| Fonction/attribut privé | préfixe `_` | `_find_project_root()` |
| Type / alias | `PascalCase` | `FeatureMatrix` |

### Règles de nommage

- Noms **explicites et orientés métier** : préférer `compute_gross_margin` à `calc_gm`.
- Éviter les abréviations non évidentes : `customer_key` ✅ plutôt que `cust_k` ❌.
- Un booléen se lit comme une question : `is_weekend`, `has_missing_values`.
- Une fonction qui a un effet de bord se nomme par un verbe d'action : `ensure_dirs_exist()`.

**Exemples valides :**

```python
RAW_DATA_DIR      # ✅ constante
get_project_version   # ✅ fonction
_find_project_root    # ✅ privé
getData           # ❌ camelCase interdit
DATA              # ❌ trop vague
```

---

## 4. Style de code & formatage

- **Ruff est l'unique outil** de lint + format (remplace Black, isort, Flake8, pyupgrade). Ne pas configurer d'autre formateur.
- **Longueur de ligne : 120 caractères.**
- **Règles activées** (voir `[tool.ruff.lint.select]`) : pyflakes (`F`), pycodestyle (`E`/`W`), bugbear (`B`), isort (`I`), pyupgrade (`UP`), simplify (`SIM`), eradicate (`ERA`), convention (`C`), pydocstyle (`D`), annotations (`ANN`).
- **Formater avant de committer** : `make format` (applique les corrections), `make lint` (vérifie sans modifier). Les hooks pre-commit le font automatiquement.
- **Ne jamais laisser de code mort commenté** : la règle `ERA` le signale. Utiliser Git pour l'historique, pas les commentaires.

```bash
make format   # corrige et reformate
make lint     # échoue si le code n'est pas propre (mode CI)
```

---

## 5. Typage statique

- **Annoter systématiquement** signatures de fonctions et constantes module-level. La règle `ANN` de Ruff le force.
- **mypy en mode `strict`** : aucune fonction non typée, aucun `Any` implicite. `ignore_missing_imports = true` pour les libs tierces sans stubs.
- **Plugin Pydantic activé** pour éviter les faux positifs sur les modèles FastAPI.
- Utiliser la syntaxe moderne (`list[str]`, `dict[str, int]`, `X | None`) — pyupgrade la garantit.

```python
def get_project_version() -> str:                     # ✅ typé
    ...

PROJECT_ROOT: Path = _find_project_root()             # ✅ constante annotée

def say_hello() -> dict[str, str]:                    # ✅ syntaxe moderne
    return {"message": "Hello world!"}
```

```bash
make typecheck   # uv run mypy --explicit-package-bases src/ app/
```

---

## 6. Documentation du code (docstrings)

Toutes les docstrings sont en **anglais**, au format **Google style** (imposé par pydocstyle via Ruff `D`).

- **Chaque module** commence par une docstring d'une ligne décrivant son rôle.
- **Chaque fonction/classe publique** documente son intention, ses arguments et sa valeur de retour.
- Documenter le *pourquoi* et les cas limites, pas la paraphrase du code.

```python
"""Centralized path management to avoid relative path hell."""

def _find_project_root() -> Path:
    """Find the project root dynamically by looking for pyproject.toml.

    Traverses up from the current file's directory until a pyproject.toml
    is found. Falls back to a fixed depth if not found.

    Returns:
        Path: The absolute path to the project root directory.

    """
    ...
```

**Exceptions tolérées** (voir `lint.ignore`) : `D107` (docstring dans `__init__`), `D203`, `D213`.

---

## 7. Configuration & secrets

- **Aucun secret dans le code ni dans Git.** Le hook `detect-private-key` et le Secret-Detection de la CI bloquent les fuites.
- **Variables d'environnement** : configuration externe (URLs de bases, clés API, credentials Snowflake) dans un fichier `.env` **non committé**. Fournir un `.env.example` documenté et à jour.
- **Chargement** : `os.getenv()` pour les cas simples, `pydantic-settings` dès qu'il y a validation ou plusieurs variables liées.
- Ne jamais logger un secret, même en debug.

```python
import os
DATABASE_URL = os.getenv("DATABASE_URL")   # ✅ depuis .env
API_KEY = "sk-prod-xxxxx"                   # ❌ jamais en dur
```

---

## 8. Logging

- Utiliser le module `logging` (ou **Loguru** pour les projets qui l'adoptent), jamais `print()` dans le code de `src/`.
- Écrire les logs dans `logs/` (git-ignoré) via `core.utils.paths.LOGS_DIR`.
- Niveaux : `DEBUG` (détail dev), `INFO` (étapes normales), `WARNING` (situation anormale récupérable), `ERROR` (échec d'une opération), `CRITICAL` (arrêt).
- Ne jamais avaler une exception silencieusement : logger et re-lever, ou gérer explicitement.

---

## 9. Gestion des données

- **`data/raw/` est immuable** : les données brutes ne sont jamais modifiées ni écrasées. Workflow imposé : `raw` → *script de nettoyage* → `interim` → `processed`. Garantit la reproductibilité depuis la source.
- **Versionnement DVC** : pour tout dataset > 10 Mo, initialiser DVC (`dvc init`) et tracker le fichier dans `data/` au lieu de le committer dans Git.
- **Notebooks = exploration uniquement.** Toute logique validée est déplacée vers `src/` où elle est typée, testée et versionnable. Les notebooks sont difficiles à tester et à diffé — `nbstripout` retire automatiquement les outputs avant commit.
- Passer par les constantes de `paths.py` pour tout accès disque.

---

## 10. Tests

- **`pytest` est le standard.** Tests dans `tests/`, organisés en `unit_test/`, `integration/`, `functional/`.
- **Couverture minimale : 80 %** sur `src/` (`--cov-fail-under=80`). La CI échoue en dessous.
- **Tests isolés et reproductibles** : utiliser les fixtures (`tmp_path`, `monkeypatch`) pour ne jamais polluer le système de fichiers réel ni dépendre de l'environnement.
- **Nommage** : `test_<comportement_attendu>`, une assertion logique par test autant que possible.
- Marquer les tests non exécutables en CI avec `@pytest.mark.ci_exclude`.

```python
def test_ensure_dirs_exist(tmp_path: Path, monkeypatch: pytest.MonkeyPatch) -> None:
    """Test that ensure_dirs_exist creates the physical directories correctly."""
    monkeypatch.setattr(paths, "RAW_DATA_DIR", tmp_path / "data" / "raw")
    paths.ensure_dirs_exist()
    assert (tmp_path / "data" / "raw").exists()
```

```bash
make test        # pytest complet avec couverture
make test-fast   # stoppe au premier échec
```

---

## 11. API (FastAPI)

- **Routes découpées par domaine** dans `api/routers/` (`greetings.py`, `system.py`…), assemblées dans `main.py`. Chaque router porte un `tags=[...]` pour la doc OpenAPI.
- **Endpoints typés** : signature et type de retour annotés (`-> dict[str, str]`), docstring courte décrivant la réponse.
- **Endpoints système obligatoires** : `/health` (monitoring, healthcheck Docker) et `/version` (traçabilité du déploiement).
- **Documentation automatique** : ne pas dupliquer à la main ce que Swagger génère à `/docs`.

```python
router = APIRouter(tags=["system"])

@router.get("/health")
def health() -> dict[str, str]:
    """Return the health status of the API."""
    return {"status": "ok"}
```

---

## 12. Git, commits & versioning

- **Conventional Commits** obligatoires : `type(scope): description` (`feat:`, `fix:`, `chore:`, `docs:`, `refactor:`, `test:`…). Permet la génération automatique du changelog et le bump sémantique.
- **commitizen** gère la version : `make bump` incrémente `VERSION` + `pyproject.toml`, met à jour `CHANGELOG.md` et crée le tag `vX.Y.Z`. Ne jamais éditer la version à la main.
- **SemVer** : `MAJOR.MINOR.PATCH`.
- **Pre-commit avant tout push** : `check-merge-conflict`, `check-added-large-files`, `end-of-file-fixer`, `nbstripout`, Ruff, mypy s'exécutent automatiquement. Installer via `make dev-install`.

```bash
git commit -m "feat(features): add rolling-window aggregations"   # ✅
git commit -m "update stuff"                                       # ❌
make bump      # incrémente version + changelog + tag
make release   # push commits + tags vers origin
```

---

## 13. CI/CD

Le pipeline (`.gitlab-ci.yml`) applique les mêmes contrôles que le local — le vert local doit produire le vert CI.

- **Stage `check`** : `python_lint` (Ruff), `python_format` (Ruff format), `python_typecheck` (mypy), `python_license_check` (`pip-licenses` — échoue sur licence GPL/copyleft), validation des messages de commit.
- **Templates sécurité GitLab inclus** : SAST, Dependency-Scanning, Secret-Detection.
- **Image de base** : image `uv` officielle alignée sur `PYTHON_VERSION`. `UV_LINK_MODE: copy` obligatoire sur les runners.
- **Reproductibilité** : `uv sync --frozen` en CI et en build Docker — jamais de résolution de dépendances non lockée.

---

## 14. Docker

- **Multi-stage** : un stage `base` (deps + user non-root) dont héritent les stages `api` et `streamlit`. Sépare build et runtime pour des images petites et sûres.
- **Utilisateur non-root** (`appuser`) obligatoire dans l'image finale.
- **Cache des dépendances** : installer `uv sync --frozen --no-install-project --no-dev` avant de copier le code, pour optimiser le cache de layers.
- **Labels OCI** : renseigner `org.opencontainers.image.*` (version, revision, date injectées au build via `make docker-build`).
- **Healthcheck** défini dans le Dockerfile, aligné sur l'endpoint `/health`.
- **`.trivyignore`** : documenter chaque CVE ignorée (raison + statut), réévaluer à chaque mise à jour de l'image de base.

---

## 15. Workflow quotidien (résumé)

```bash
make dev-install         # 1. setup initial (deps + hooks)
# ... développement ...
make format              # 2. formater
make check               # 3. lint + typecheck + tests en local
git commit -m "feat: ..." # 4. commit conventionnel (hooks vérifient)
make bump                # 5. (au moment d'une release) version + changelog
make release             # 6. push commits + tags
```

---

## 16. Checklist avant merge / release

**Obligatoire :**
- [ ] Le code passe `make check` (lint + typecheck + tests) en local.
- [ ] Couverture ≥ 80 % sur `src/`.
- [ ] Toutes les fonctions/classes publiques sont typées et ont une docstring anglaise.
- [ ] Aucun secret ni clé en dur ; `.env.example` à jour.
- [ ] Aucun chemin relatif en dur (usage de `core.utils.paths`).
- [ ] Aucune logique métier laissée dans un notebook.
- [ ] Les données brutes (`data/raw/`) n'ont pas été modifiées ; gros fichiers trackés via DVC.
- [ ] Messages de commit au format Conventional Commits.
- [ ] `uv.lock` committé et à jour.
- [ ] Pipeline CI vert (check + sécurité).

**Optionnel (projets conteneurisés) :**
- [ ] Image Docker multi-stage, utilisateur non-root, labels OCI renseignés.
- [ ] Healthcheck fonctionnel sur `/health`.
- [ ] CVE résiduelles documentées dans `.trivyignore`.
