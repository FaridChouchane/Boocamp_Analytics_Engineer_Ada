# 🧪 Cours Complet : Les Tests en Python avec pytest

> **De débutant à professionnel — Un guide progressif, concret et actionnable.**
>
> *Par un Senior Python Engineer — pour ceux qui veulent coder comme des pros.*

---

## Table des matières

1. [Introduction aux tests](#1-introduction-aux-tests)
2. [Premiers pas avec pytest](#2-premiers-pas-avec-pytest)
3. [Comprendre les assertions](#3-comprendre-les-assertions)
4. [Organisation des tests](#4-organisation-des-tests)
5. [Les Fixtures — Le cœur de pytest](#5-les-fixtures--le-cœur-de-pytest)
6. [Paramétrisation des tests](#6-paramétrisation-des-tests)
7. [Tester une classe — Cas réel](#7-tester-une-classe--cas-réel)
8. [Gestion des erreurs et exceptions](#8-gestion-des-erreurs-et-exceptions)
9. [Bonnes pratiques PRO](#9-bonnes-pratiques-pro)
10. [Pièges fréquents](#10-pièges-fréquents)
11. [Mini projet guidé — TaskManager](#11-mini-projet-guidé--taskmanager)
12. [Structure d'un projet professionnel](#12-structure-dun-projet-professionnel)
13. [Intégration avec les outils modernes](#13-intégration-avec-les-outils-modernes)

---

## 1. Introduction aux tests

### 1.1 Pourquoi tester ?

**Analogie simple :** Imagine que tu construis un pont. Est-ce que tu laisserais les voitures passer dessus sans vérifier qu'il tient ? Non. Les tests, c'est exactement ça : **vérifier que ton code tient debout avant de le mettre en production.**

Concrètement, les tests te permettent de :

- **Détecter les bugs tôt** — Un bug trouvé pendant le développement coûte 10x moins cher qu'un bug trouvé en production.
- **Refactorer sans peur** — Tu veux améliorer ton code ? Si tes tests passent après la modification, tu sais que tu n'as rien cassé.
- **Documenter ton code** — Un bon test montre exactement comment une fonction doit être utilisée.
- **Gagner en confiance** — Déployer du code testé, c'est dormir tranquille.

**Conseil terrain :** Dans le monde professionnel, un code sans tests est considéré comme du code non terminé. Les entreprises sérieuses exigent des tests. C'est un critère d'embauche.

### 1.2 Les types de tests

Il existe trois grandes familles de tests, chacune avec un rôle précis :

**Tests unitaires** — Tester une seule fonction ou méthode, isolément. C'est comme vérifier qu'une seule brique est solide.

```python
def add(a, b):
    return a + b

# Test unitaire : on vérifie UNE fonction
assert add(2, 3) == 5
```

**Tests d'intégration** — Tester que plusieurs composants fonctionnent bien ensemble. C'est comme vérifier que les briques tiennent bien quand on les empile.

```python
# Exemple : tester qu'une fonction qui lit un fichier 
# ET traite les données fonctionne correctement ensemble
def read_and_process(filepath):
    data = read_csv(filepath)
    return transform(data)
```

**Tests end-to-end (e2e)** — Tester l'application complète du point de vue de l'utilisateur. C'est comme vérifier que le pont entier tient avec des voitures dessus.

```python
# Exemple : tester une API complète
# 1. Envoyer une requête HTTP
# 2. Vérifier la réponse
# 3. Vérifier que la base de données a été mise à jour
```

**La pyramide des tests :**

```
         /\
        /  \        Tests e2e (peu, lents, coûteux)
       /    \
      /------\
     /        \     Tests d'intégration (moyens)
    /          \
   /------------\
  /              \  Tests unitaires (beaucoup, rapides, simples)
 /________________\
```

> 🧠 **À retenir :** En pratique, 70-80% de tes tests seront des tests unitaires. C'est le meilleur rapport effort/valeur. Ce cours se concentre principalement sur les tests unitaires avec pytest.

### 1.3 Ton premier test — Sans pytest

Avant d'utiliser pytest, comprenons le mécanisme de base. Python a un mot-clé natif : `assert`.

```python
# fichier: calculatrice.py
def add(a, b):
    return a + b

def subtract(a, b):
    return a - b

def multiply(a, b):
    return a * b

# Tests manuels avec assert
assert add(2, 3) == 5, "2 + 3 devrait faire 5"
assert add(-1, 1) == 0, "-1 + 1 devrait faire 0"
assert add(0, 0) == 0, "0 + 0 devrait faire 0"

assert subtract(10, 3) == 7, "10 - 3 devrait faire 7"
assert multiply(4, 5) == 20, "4 * 5 devrait faire 20"

print("✅ Tous les tests passent !")
```

**Exécution :**

```bash
$ python calculatrice.py
✅ Tous les tests passent !
```

Si un test échoue :

```python
assert add(2, 3) == 6, "2 + 3 devrait faire 6"
# AssertionError: 2 + 3 devrait faire 6
```

**Problème avec cette approche :**
- Pas de rapport clair sur ce qui passe / échoue
- Si un `assert` échoue, les suivants ne s'exécutent pas
- Pas de mécanisme de setup/teardown
- Pas de réutilisation facile

C'est là que **pytest** entre en jeu.

### 📝 Mini exercice 1

Crée un fichier `math_basics.py` avec ces fonctions :

```python
def divide(a, b):
    if b == 0:
        raise ValueError("Division par zéro impossible")
    return a / b

def is_even(n):
    return n % 2 == 0
```

Écris 3 tests manuels avec `assert` pour chaque fonction (6 tests au total). Vérifie que tout passe en exécutant le fichier.

---

## 2. Premiers pas avec pytest

### 2.1 Installation

```bash
# Avec pip (méthode standard)
pip install pytest

# Vérifier l'installation
pytest --version
```

Tu devrais voir quelque chose comme :

```
pytest 8.x.x
```

**Conseil terrain :** Dans un vrai projet, mets toujours pytest dans ton fichier `requirements.txt` ou `requirements-dev.txt` :

```
# requirements-dev.txt
pytest==8.3.4
pytest-cov==6.0.0
```

### 2.2 Structure d'un test pytest

Un test pytest suit une convention simple :

1. Le fichier doit commencer par `test_` ou finir par `_test.py`
2. Chaque fonction de test doit commencer par `test_`
3. On utilise `assert` pour vérifier les résultats

**Analogie :** Pense à pytest comme un inspecteur qui fait sa ronde. Il entre dans chaque pièce dont le nom commence par "test_", vérifie tout ce qui est marqué "assert", et te fait un rapport à la fin.

### 2.3 Ton premier test pytest

**Le code à tester** — `calculator.py` :

```python
def add(a, b):
    """Additionne deux nombres."""
    return a + b

def subtract(a, b):
    """Soustrait b de a."""
    return a - b

def multiply(a, b):
    """Multiplie deux nombres."""
    return a * b

def divide(a, b):
    """Divise a par b."""
    if b == 0:
        raise ValueError("Division par zéro impossible")
    return a / b
```

**Le fichier de test** — `test_calculator.py` :

```python
from calculator import add, subtract, multiply, divide

def test_add_positive_numbers():
    """Teste l'addition de deux nombres positifs."""
    result = add(2, 3)
    assert result == 5

def test_add_negative_numbers():
    """Teste l'addition avec des nombres négatifs."""
    assert add(-1, -1) == -2

def test_add_zero():
    """Teste l'addition avec zéro."""
    assert add(5, 0) == 5

def test_subtract():
    """Teste la soustraction."""
    assert subtract(10, 3) == 7

def test_multiply():
    """Teste la multiplication."""
    assert multiply(4, 5) == 20
```

**Exécution :**

```bash
# Lancer tous les tests
$ pytest

# Résultat attendu :
========================= test session starts =========================
collected 5 items

test_calculator.py .....                                         [100%]

========================= 5 passed in 0.01s ==========================
```

Chaque `.` représente un test qui passe. Si un test échoue, tu verras un `F` avec le détail de l'erreur.

### 2.4 Les options utiles de pytest

```bash
# Mode verbeux — voir le nom de chaque test
$ pytest -v

test_calculator.py::test_add_positive_numbers PASSED
test_calculator.py::test_add_negative_numbers PASSED
test_calculator.py::test_add_zero PASSED
test_calculator.py::test_subtract PASSED
test_calculator.py::test_multiply PASSED

# Lancer un seul fichier
$ pytest test_calculator.py

# Lancer un seul test
$ pytest test_calculator.py::test_add_positive_numbers

# Lancer les tests dont le nom contient "add"
$ pytest -k "add"

# Arrêter au premier échec
$ pytest -x

# Afficher les print() dans les tests
$ pytest -s

# Combiner les options
$ pytest -v -x -s
```

> 🧠 **À retenir :** `pytest -v` est ta commande quotidienne. `-x` est utile pour le debug. `-k` est parfait pour travailler sur une fonctionnalité précise.

### 2.5 Convention de nommage — Les règles d'or

```
✅ BON                          ❌ MAUVAIS
─────────────────────────────    ─────────────────────────────
test_calculator.py               calculator_test_file.py
test_add_two_numbers()           testAdd()
test_user_creation_success()     test1()
test_divide_by_zero_raises()     test_it_works()
```

**Règle simple :** Le nom du test doit répondre à la question "Que teste-t-on et quel résultat attend-on ?"

- `test_add_positive_numbers` → On teste l'addition de nombres positifs
- `test_divide_by_zero_raises_error` → On teste que la division par zéro lève une erreur
- `test_user_without_email_is_invalid` → On teste qu'un utilisateur sans email est invalide

### 📝 Mini exercice 2

Reprends les fonctions `divide` et `is_even` de l'exercice 1. Crée un fichier `test_math_basics.py` avec pytest :

1. Écris 3 tests pour `divide` (cas normal, résultat décimal, cas limite)
2. Écris 3 tests pour `is_even` (pair, impair, zéro)
3. Lance `pytest -v` et vérifie que tout passe.

---

## 3. Comprendre les assertions

### 3.1 Le assert de pytest — Plus puissant qu'il n'y paraît

Avec pytest, un simple `assert` donne des messages d'erreur **incroyablement détaillés**. C'est un des gros avantages de pytest par rapport à `unittest`.

```python
def test_assertion_example():
    result = [1, 2, 3, 4]
    expected = [1, 2, 4, 4]
    assert result == expected
```

**Sortie d'erreur pytest :**

```
FAILED test_example.py::test_assertion_example
    def test_assertion_example():
        result = [1, 2, 3, 4]
        expected = [1, 2, 4, 4]
>       assert result == expected
E       AssertionError: assert [1, 2, 3, 4] == [1, 2, 4, 4]
E         At index 2 diff: 3 != 4
```

Pytest te montre **exactement** où est la différence. Pas besoin de méthodes spéciales comme `assertEqual`, `assertIn`, etc.

### 3.2 Tous les types d'assertions courantes

```python
# --- Égalité ---
def test_equality():
    assert 1 + 1 == 2
    assert "hello".upper() == "HELLO"
    assert [1, 2, 3] == [1, 2, 3]

# --- Vérité / Fausseté ---
def test_truthiness():
    assert True
    assert 1  # tout nombre non-nul est "truthy"
    assert "non-vide"  # toute string non-vide est "truthy"
    assert [1]  # toute liste non-vide est "truthy"
    
    assert not False
    assert not 0
    assert not ""
    assert not []

# --- Appartenance ---
def test_membership():
    fruits = ["pomme", "banane", "cerise"]
    assert "pomme" in fruits
    assert "kiwi" not in fruits
    
    text = "Bonjour le monde"
    assert "monde" in text

# --- Comparaisons ---
def test_comparisons():
    assert 10 > 5
    assert 3 <= 3
    assert 7 >= 7
    assert 1 != 2

# --- Type ---
def test_types():
    assert isinstance(42, int)
    assert isinstance("hello", str)
    assert isinstance([1, 2], list)
    assert isinstance({"a": 1}, dict)

# --- None ---
def test_none():
    result = None
    assert result is None
    
    other = "something"
    assert other is not None

# --- Approximation pour les flottants ---
def test_floats():
    # ❌ DANGEREUX — les flottants ne sont pas exacts
    # assert 0.1 + 0.2 == 0.3  # FAIL !
    
    # ✅ CORRECT — utilise pytest.approx
    import pytest
    assert 0.1 + 0.2 == pytest.approx(0.3)
    assert 2.0000001 == pytest.approx(2.0, abs=1e-6)
```

### 3.3 Messages d'erreur personnalisés

Tu peux ajouter un message pour clarifier l'intention d'un test :

```python
def test_user_age_is_valid():
    age = -5
    assert age >= 0, f"L'âge doit être positif, mais on a reçu {age}"
```

**Sortie :**

```
AssertionError: L'âge doit être positif, mais on a reçu -5
```

**Conseil terrain :** Utilise des messages personnalisés uniquement quand le test n'est pas évident. Si le nom du test est bien choisi, le message par défaut de pytest suffit souvent.

### 3.4 Bonnes pratiques des assertions

```python
# ✅ BON — Un assert principal par test
def test_user_full_name():
    user = create_user("Jean", "Dupont")
    assert user.full_name == "Jean Dupont"

# ❌ MAUVAIS — Trop d'assertions non liées dans un seul test
def test_user_everything():
    user = create_user("Jean", "Dupont")
    assert user.full_name == "Jean Dupont"
    assert user.email is not None
    assert user.age >= 0
    assert user.is_active == True
    # Si le premier assert échoue, on ne vérifie jamais les autres

# ✅ BON — Plusieurs assertions LIÉES (ça va)
def test_user_creation_sets_defaults():
    user = create_user("Jean", "Dupont")
    assert user.is_active == True
    assert user.role == "member"
    # Ces deux assertions testent le même comportement : les valeurs par défaut
```

> 🧠 **À retenir :** La règle n'est pas "1 assert par test" mais "1 comportement par test". Plusieurs `assert` sont OK s'ils vérifient le même comportement. Un seul `assert` par test est idéal quand c'est possible.

### 📝 Mini exercice 3

Crée un fichier `test_assertions.py` avec ces tests :

```python
def test_string_operations():
    """Teste que 'hello world'.split() retourne ['hello', 'world']"""
    # À compléter

def test_dictionary_access():
    """Teste qu'un dictionnaire contient les bonnes clés et valeurs"""
    config = {"host": "localhost", "port": 8080, "debug": True}
    # Écris 3 assertions : clé existe, valeur correcte, type correct

def test_list_sorting():
    """Teste que sorted() trie correctement une liste"""
    # À compléter avec une liste de nombres et une liste de strings

def test_float_calculation():
    """Teste un calcul avec des flottants en utilisant pytest.approx"""
    # À compléter
```

---

## 4. Organisation des tests

### 4.1 Structure de base

**Analogie :** Imagine ton projet comme une maison. Le code source est dans les chambres, et les tests sont dans les pièces de vérification correspondantes. Chaque chambre a sa pièce de vérification attitrée.

**Structure minimale (petit projet) :**

```
mon_projet/
├── calculator.py
├── user.py
├── test_calculator.py
└── test_user.py
```

**Structure recommandée (projet moyen) :**

```
mon_projet/
├── src/
│   ├── __init__.py
│   ├── calculator.py
│   └── user.py
├── tests/
│   ├── __init__.py
│   ├── conftest.py          ← fixtures partagées (on verra ça après)
│   ├── test_calculator.py
│   └── test_user.py
├── requirements.txt
└── README.md
```

**Structure pro (grand projet) :**

```
mon_projet/
├── src/
│   ├── __init__.py
│   ├── models/
│   │   ├── __init__.py
│   │   ├── user.py
│   │   └── task.py
│   ├── services/
│   │   ├── __init__.py
│   │   ├── auth_service.py
│   │   └── task_service.py
│   └── utils/
│       ├── __init__.py
│       └── validators.py
├── tests/
│   ├── __init__.py
│   ├── conftest.py
│   ├── unit/
│   │   ├── __init__.py
│   │   ├── models/
│   │   │   ├── test_user.py
│   │   │   └── test_task.py
│   │   └── services/
│   │       ├── test_auth_service.py
│   │       └── test_task_service.py
│   └── integration/
│       ├── __init__.py
│       └── test_api.py
├── requirements.txt
├── requirements-dev.txt
└── pytest.ini
```

### 4.2 Le fichier `conftest.py`

C'est un fichier **spécial** reconnu automatiquement par pytest. Il contient des fixtures (on verra en détail plus tard) partagées entre tous les tests du même dossier.

```python
# tests/conftest.py
import pytest

@pytest.fixture
def sample_user():
    """Crée un utilisateur de test réutilisable."""
    return {"name": "Alice", "email": "alice@example.com", "age": 30}

@pytest.fixture
def empty_list():
    """Retourne une liste vide pour les tests."""
    return []
```

**Règles du `conftest.py` :**
- Pas besoin de l'importer — pytest le découvre automatiquement
- Il s'applique à tous les tests dans le même dossier et sous-dossiers
- Tu peux avoir plusieurs `conftest.py` à différents niveaux

### 4.3 Configuration avec `pytest.ini` ou `pyproject.toml`

```ini
# pytest.ini (à la racine du projet)
[pytest]
testpaths = tests
python_files = test_*.py
python_functions = test_*
addopts = -v --tb=short
```

Ou avec `pyproject.toml` (plus moderne) :

```toml
# pyproject.toml
[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py"]
python_functions = ["test_*"]
addopts = "-v --tb=short"
```

> 🧠 **À retenir :** Commence simple. Un dossier `tests/` avec un `conftest.py` suffit pour 90% des projets. La structure grandit avec le projet.

### 📝 Mini exercice 4

Crée cette structure de projet :

```
mini_project/
├── src/
│   ├── __init__.py
│   └── string_utils.py     ← écris 3 fonctions de manipulation de strings
├── tests/
│   ├── __init__.py
│   ├── conftest.py          ← une fixture qui retourne une phrase de test
│   └── test_string_utils.py ← 6 tests minimum
└── pytest.ini               ← configure testpaths
```

Lance `pytest -v` depuis la racine et vérifie que tout passe.

---

## 5. Les Fixtures — Le cœur de pytest

### 5.1 C'est quoi une fixture ?

**Analogie :** Tu es cuisinier. Avant chaque recette, tu dois préparer tes ingrédients : laver les légumes, mesurer la farine, préchauffer le four. C'est la **mise en place**. En cuisine, tu ne recommences pas tout à zéro pour chaque plat — tu prépares une fois, tu réutilises.

**Une fixture, c'est exactement ça : une "mise en place" pour tes tests.** C'est du code qui prépare les données ou les objets dont tes tests ont besoin.

### 5.2 Pourquoi c'est puissant ?

Sans fixtures, tu répètes le même code partout :

```python
# ❌ SANS fixture — beaucoup de répétition
def test_user_full_name():
    user = User(first_name="Jean", last_name="Dupont", email="jean@mail.com")
    assert user.full_name == "Jean Dupont"

def test_user_email():
    user = User(first_name="Jean", last_name="Dupont", email="jean@mail.com")
    assert user.email == "jean@mail.com"

def test_user_is_active():
    user = User(first_name="Jean", last_name="Dupont", email="jean@mail.com")
    assert user.is_active == True
```

Avec fixtures :

```python
# ✅ AVEC fixture — propre et réutilisable
import pytest

@pytest.fixture
def user():
    return User(first_name="Jean", last_name="Dupont", email="jean@mail.com")

def test_user_full_name(user):  # pytest injecte automatiquement la fixture
    assert user.full_name == "Jean Dupont"

def test_user_email(user):
    assert user.email == "jean@mail.com"

def test_user_is_active(user):
    assert user.is_active == True
```

**La magie :** Tu déclares la fixture comme paramètre de ta fonction de test, et pytest l'injecte automatiquement. Pas besoin d'appeler la fixture toi-même.

### 5.3 Fixtures de base

```python
import pytest

# --- Fixture simple : retourne une valeur ---
@pytest.fixture
def sample_name():
    return "Alice"

def test_greeting(sample_name):
    assert f"Bonjour {sample_name}" == "Bonjour Alice"


# --- Fixture avec un objet ---
@pytest.fixture
def config():
    return {
        "database": "test_db",
        "host": "localhost",
        "port": 5432,
        "debug": True
    }

def test_config_has_database(config):
    assert "database" in config
    assert config["database"] == "test_db"

def test_config_debug_mode(config):
    assert config["debug"] is True
```

### 5.4 Fixtures avec setup ET teardown

Parfois, tu dois préparer quelque chose ET nettoyer après. C'est le cas avec les fichiers, les connexions réseau, les bases de données.

```python
import pytest
import os

@pytest.fixture
def temp_file():
    """Crée un fichier temporaire, puis le supprime après le test."""
    # --- SETUP (avant le test) ---
    filepath = "test_data.txt"
    with open(filepath, "w") as f:
        f.write("données de test")
    
    yield filepath  # ← Le test s'exécute ici, avec filepath comme valeur
    
    # --- TEARDOWN (après le test, même si le test échoue) ---
    if os.path.exists(filepath):
        os.remove(filepath)

def test_file_exists(temp_file):
    assert os.path.exists(temp_file)

def test_file_content(temp_file):
    with open(temp_file) as f:
        content = f.read()
    assert content == "données de test"
```

**Le mot-clé `yield` :** Tout ce qui est avant `yield` est le setup. Tout ce qui est après `yield` est le teardown. Le teardown s'exécute **toujours**, même si le test échoue.

### 5.5 Portée des fixtures (scope)

Par défaut, une fixture est recréée pour **chaque test**. Mais tu peux changer ça :

```python
import pytest

# Recréée pour CHAQUE test (défaut)
@pytest.fixture(scope="function")
def fresh_list():
    return [1, 2, 3]

# Créée une seule fois pour tout le MODULE (fichier)
@pytest.fixture(scope="module")
def database_connection():
    conn = create_connection("test_db")
    yield conn
    conn.close()

# Créée une seule fois pour toute la SESSION de tests
@pytest.fixture(scope="session")
def app_config():
    return load_config("test_config.yaml")
```

**Quand utiliser quel scope ?**

| Scope | Quand l'utiliser | Exemple |
|-------|-----------------|---------|
| `function` (défaut) | Données qui doivent être fraîches à chaque test | Objet utilisateur, liste |
| `module` | Ressource coûteuse, partagée dans un fichier | Connexion DB |
| `session` | Ressource globale, partagée partout | Configuration app |

### 5.6 Fixtures qui utilisent d'autres fixtures

Les fixtures peuvent dépendre les unes des autres :

```python
import pytest

@pytest.fixture
def database_url():
    return "postgresql://localhost:5432/test_db"

@pytest.fixture
def database_connection(database_url):
    """Cette fixture utilise la fixture database_url."""
    conn = connect(database_url)
    yield conn
    conn.close()

@pytest.fixture
def user_repository(database_connection):
    """Cette fixture utilise la fixture database_connection."""
    return UserRepository(database_connection)

def test_create_user(user_repository):
    """Ce test utilise la chaîne complète de fixtures."""
    user = user_repository.create(name="Alice")
    assert user.name == "Alice"
```

**Analogie :** C'est comme une chaîne de montage. Chaque étape dépend de la précédente, et pytest gère tout automatiquement.

### 5.7 Fixtures dans conftest.py — Partage entre fichiers

```python
# tests/conftest.py — accessible par TOUS les tests
import pytest

@pytest.fixture
def sample_users():
    """Liste d'utilisateurs pour les tests."""
    return [
        {"name": "Alice", "role": "admin"},
        {"name": "Bob", "role": "member"},
        {"name": "Charlie", "role": "member"},
    ]

@pytest.fixture
def admin_user(sample_users):
    """Retourne le premier admin trouvé."""
    for user in sample_users:
        if user["role"] == "admin":
            return user
    return None
```

```python
# tests/test_permissions.py — utilise les fixtures de conftest.py
def test_admin_has_admin_role(admin_user):
    assert admin_user["role"] == "admin"

def test_sample_users_count(sample_users):
    assert len(sample_users) == 3
```

### 5.8 Exemple avancé — Fixture réaliste

```python
# tests/conftest.py
import pytest
import json
import os

class FakeDatabase:
    """Simule une base de données en mémoire."""
    def __init__(self):
        self._data = {}
        self._next_id = 1
    
    def insert(self, table, record):
        record_id = self._next_id
        self._next_id += 1
        record["id"] = record_id
        self._data.setdefault(table, []).append(record)
        return record
    
    def find_all(self, table):
        return self._data.get(table, [])
    
    def find_by_id(self, table, record_id):
        for record in self._data.get(table, []):
            if record["id"] == record_id:
                return record
        return None
    
    def clear(self):
        self._data.clear()
        self._next_id = 1

@pytest.fixture
def db():
    """Base de données fraîche pour chaque test."""
    database = FakeDatabase()
    yield database
    database.clear()  # Nettoyage après le test

@pytest.fixture
def db_with_users(db):
    """Base avec des utilisateurs pré-remplis."""
    db.insert("users", {"name": "Alice", "email": "alice@test.com"})
    db.insert("users", {"name": "Bob", "email": "bob@test.com"})
    return db
```

```python
# tests/test_database.py
def test_insert_and_find(db):
    db.insert("products", {"name": "Laptop", "price": 999})
    products = db.find_all("products")
    assert len(products) == 1
    assert products[0]["name"] == "Laptop"

def test_find_by_id(db):
    record = db.insert("products", {"name": "Phone", "price": 499})
    found = db.find_by_id("products", record["id"])
    assert found is not None
    assert found["name"] == "Phone"

def test_db_with_preloaded_users(db_with_users):
    users = db_with_users.find_all("users")
    assert len(users) == 2
    assert users[0]["name"] == "Alice"

def test_each_test_gets_fresh_db(db):
    """Vérifie que la DB est vide (pas polluée par un autre test)."""
    users = db.find_all("users")
    assert len(users) == 0
```

> 🧠 **À retenir :** Les fixtures sont LE concept le plus important de pytest. Maîtrise-les, et tu maîtriseras pytest. Commence par des fixtures simples (retourner une valeur), puis avance vers yield (setup/teardown) et les scopes.

### 📝 Mini exercice 5

1. Crée une fixture `shopping_cart` qui retourne une liste vide.
2. Crée une fixture `shopping_cart_with_items` qui utilise `shopping_cart` et y ajoute 3 articles.
3. Écris ces tests :
   - `test_empty_cart_has_zero_items`
   - `test_cart_with_items_has_three_items`
   - `test_add_item_to_cart`
   - `test_remove_item_from_cart`

---

## 6. Paramétrisation des tests

### 6.1 Le problème

Souvent, tu veux tester la même logique avec plusieurs jeux de données :

```python
# ❌ Répétitif et ennuyeux
def test_add_positive():
    assert add(2, 3) == 5

def test_add_negative():
    assert add(-1, -1) == -2

def test_add_zero():
    assert add(0, 0) == 0

def test_add_mixed():
    assert add(-1, 1) == 0
```

### 6.2 La solution — `@pytest.mark.parametrize`

```python
import pytest
from calculator import add

@pytest.mark.parametrize("a, b, expected", [
    (2, 3, 5),          # cas normal
    (-1, -1, -2),       # nombres négatifs
    (0, 0, 0),          # zéros
    (-1, 1, 0),         # mixte
    (100, 200, 300),    # grands nombres
    (0.1, 0.2, 0.3),   # flottants (attention aux approximations)
])
def test_add(a, b, expected):
    assert add(a, b) == pytest.approx(expected)
```

**Exécution avec `-v` :**

```
test_calculator.py::test_add[2-3-5] PASSED
test_calculator.py::test_add[-1--1--2] PASSED
test_calculator.py::test_add[0-0-0] PASSED
test_calculator.py::test_add[-1-1-0] PASSED
test_calculator.py::test_add[100-200-300] PASSED
test_calculator.py::test_add[0.1-0.2-0.3] PASSED
```

**6 tests en une seule fonction !** Chaque combinaison de paramètres crée un test indépendant.

### 6.3 Avec des identifiants lisibles

```python
import pytest

@pytest.mark.parametrize("email, is_valid", [
    pytest.param("user@example.com", True, id="email_valide_standard"),
    pytest.param("user@sub.domain.com", True, id="email_avec_sous_domaine"),
    pytest.param("user@.com", False, id="domaine_commence_par_point"),
    pytest.param("@example.com", False, id="pas_de_nom_utilisateur"),
    pytest.param("user@", False, id="pas_de_domaine"),
    pytest.param("", False, id="chaine_vide"),
])
def test_email_validation(email, is_valid):
    assert validate_email(email) == is_valid
```

**Sortie :**

```
test_email.py::test_email_validation[email_valide_standard] PASSED
test_email.py::test_email_validation[email_avec_sous_domaine] PASSED
test_email.py::test_email_validation[domaine_commence_par_point] PASSED
...
```

### 6.4 Combinaison de paramètres

Tu peux empiler les décorateurs pour tester toutes les combinaisons :

```python
@pytest.mark.parametrize("x", [1, 2, 3])
@pytest.mark.parametrize("y", [10, 20])
def test_multiply(x, y):
    result = x * y
    assert result == x * y
    # Crée 6 tests : (1,10), (1,20), (2,10), (2,20), (3,10), (3,20)
```

### 6.5 Paramétriser avec des fixtures

```python
import pytest

@pytest.fixture(params=["sqlite", "postgresql", "mysql"])
def database_type(request):
    """Cette fixture sera appelée 3 fois, une pour chaque base."""
    return request.param

def test_connection_string(database_type):
    """Ce test sera exécuté 3 fois automatiquement."""
    conn_string = build_connection_string(database_type)
    assert database_type in conn_string
```

> 🧠 **À retenir :** La paramétrisation élimine la duplication. Utilise `@pytest.mark.parametrize` quand tu testes la même logique avec des données différentes. C'est un outil que les développeurs seniors utilisent massivement.

### 📝 Mini exercice 6

Crée une fonction `classify_age(age)` qui retourne :
- `"enfant"` si age < 13
- `"adolescent"` si 13 <= age < 18
- `"adulte"` si 18 <= age < 65
- `"senior"` si age >= 65

Écris un test paramétré qui couvre au moins 8 cas (inclus les cas limites : 0, 12, 13, 17, 18, 64, 65, 100).

---

## 7. Tester une classe — Cas réel

### 7.1 La classe `TaskManager`

Passons à un exemple concret et réaliste. Voici une classe de gestion de tâches :

```python
# src/task_manager.py
from datetime import datetime, timedelta
from typing import Optional


class Task:
    """Représente une tâche."""
    
    def __init__(self, title: str, owner: str, priority: str = "medium"):
        if not title.strip():
            raise ValueError("Le titre ne peut pas être vide")
        if priority not in ("low", "medium", "high", "critical"):
            raise ValueError(f"Priorité invalide : {priority}")
        
        self.title = title
        self.owner = owner
        self.priority = priority
        self.status = "todo"
        self.created_at = datetime.now()
        self.completed_at: Optional[datetime] = None
    
    def complete(self):
        """Marque la tâche comme terminée."""
        if self.status == "completed":
            raise ValueError("La tâche est déjà terminée")
        self.status = "completed"
        self.completed_at = datetime.now()
    
    def reopen(self):
        """Rouvre une tâche terminée."""
        if self.status != "completed":
            raise ValueError("Seule une tâche terminée peut être rouverte")
        self.status = "todo"
        self.completed_at = None
    
    def __repr__(self):
        return f"Task('{self.title}', owner='{self.owner}', status='{self.status}')"


class TaskManager:
    """Gestionnaire de tâches avec support multi-utilisateurs."""
    
    def __init__(self, current_user: str):
        if not current_user.strip():
            raise ValueError("Le nom d'utilisateur ne peut pas être vide")
        self.current_user = current_user
        self._tasks: list[Task] = []
    
    def add_task(self, title: str, owner: Optional[str] = None,
                 priority: str = "medium") -> Task:
        """
        Ajoute une tâche.
        Si owner n'est pas spécifié, le current_user devient le owner.
        """
        actual_owner = owner if owner else self.current_user
        task = Task(title=title, owner=actual_owner, priority=priority)
        self._tasks.append(task)
        return task
    
    def get_tasks(self, status: Optional[str] = None,
                  owner: Optional[str] = None) -> list[Task]:
        """Filtre les tâches par statut et/ou propriétaire."""
        result = self._tasks
        if status:
            result = [t for t in result if t.status == status]
        if owner:
            result = [t for t in result if t.owner == owner]
        return result
    
    def get_my_tasks(self) -> list[Task]:
        """Retourne les tâches du current_user."""
        return self.get_tasks(owner=self.current_user)
    
    def complete_task(self, title: str) -> Task:
        """Marque une tâche comme terminée par son titre."""
        task = self._find_task(title)
        if not task:
            raise ValueError(f"Tâche non trouvée : {title}")
        task.complete()
        return task
    
    def count_by_status(self) -> dict[str, int]:
        """Compte les tâches par statut."""
        counts = {"todo": 0, "completed": 0}
        for task in self._tasks:
            counts[task.status] = counts.get(task.status, 0) + 1
        return counts
    
    def get_high_priority_tasks(self) -> list[Task]:
        """Retourne les tâches high et critical non terminées."""
        return [
            t for t in self._tasks
            if t.priority in ("high", "critical") and t.status != "completed"
        ]
    
    def _find_task(self, title: str) -> Optional[Task]:
        """Cherche une tâche par titre."""
        for task in self._tasks:
            if task.title == title:
                return task
        return None
```

### 7.2 Les tests complets

```python
# tests/test_task_manager.py
import pytest
from src.task_manager import Task, TaskManager


# ═══════════════════════════════════════════
# FIXTURES
# ═══════════════════════════════════════════

@pytest.fixture
def manager():
    """TaskManager frais avec un utilisateur par défaut."""
    return TaskManager(current_user="alice")

@pytest.fixture
def manager_with_tasks(manager):
    """TaskManager avec des tâches pré-remplies."""
    manager.add_task("Écrire les tests", priority="high")
    manager.add_task("Relire le code", owner="bob", priority="medium")
    manager.add_task("Déployer en prod", priority="critical")
    manager.add_task("Mettre à jour le README", priority="low")
    return manager


# ═══════════════════════════════════════════
# TESTS DE LA CLASSE Task
# ═══════════════════════════════════════════

class TestTask:
    """Tests pour la classe Task."""
    
    def test_creation_with_defaults(self):
        task = Task(title="Ma tâche", owner="alice")
        assert task.title == "Ma tâche"
        assert task.owner == "alice"
        assert task.priority == "medium"
        assert task.status == "todo"
        assert task.completed_at is None
    
    def test_creation_with_custom_priority(self):
        task = Task(title="Urgent", owner="alice", priority="critical")
        assert task.priority == "critical"
    
    def test_creation_with_empty_title_raises(self):
        with pytest.raises(ValueError, match="titre ne peut pas être vide"):
            Task(title="", owner="alice")
    
    def test_creation_with_invalid_priority_raises(self):
        with pytest.raises(ValueError, match="Priorité invalide"):
            Task(title="Test", owner="alice", priority="ultra")
    
    def test_complete_changes_status(self):
        task = Task(title="À faire", owner="alice")
        task.complete()
        assert task.status == "completed"
        assert task.completed_at is not None
    
    def test_complete_already_completed_raises(self):
        task = Task(title="Finie", owner="alice")
        task.complete()
        with pytest.raises(ValueError, match="déjà terminée"):
            task.complete()
    
    def test_reopen_completed_task(self):
        task = Task(title="Finie", owner="alice")
        task.complete()
        task.reopen()
        assert task.status == "todo"
        assert task.completed_at is None
    
    def test_reopen_non_completed_raises(self):
        task = Task(title="En cours", owner="alice")
        with pytest.raises(ValueError, match="tâche terminée"):
            task.reopen()


# ═══════════════════════════════════════════
# TESTS DE LA CLASSE TaskManager
# ═══════════════════════════════════════════

class TestTaskManagerCreation:
    """Tests pour la création du TaskManager."""
    
    def test_creation_with_valid_user(self):
        tm = TaskManager(current_user="alice")
        assert tm.current_user == "alice"
    
    def test_creation_with_empty_user_raises(self):
        with pytest.raises(ValueError):
            TaskManager(current_user="")
    
    def test_new_manager_has_no_tasks(self, manager):
        assert len(manager.get_tasks()) == 0


class TestTaskManagerAddTask:
    """Tests pour l'ajout de tâches."""
    
    def test_add_task_basic(self, manager):
        task = manager.add_task("Nouvelle tâche")
        assert task.title == "Nouvelle tâche"
        assert len(manager.get_tasks()) == 1
    
    def test_add_task_without_owner_uses_current_user(self, manager):
        """Test du owner fallback — si pas de owner, c'est current_user."""
        task = manager.add_task("Ma tâche")
        assert task.owner == "alice"  # current_user
    
    def test_add_task_with_explicit_owner(self, manager):
        task = manager.add_task("Tâche de Bob", owner="bob")
        assert task.owner == "bob"
    
    def test_add_task_with_priority(self, manager):
        task = manager.add_task("Urgent", priority="high")
        assert task.priority == "high"
    
    def test_add_multiple_tasks(self, manager):
        manager.add_task("Tâche 1")
        manager.add_task("Tâche 2")
        manager.add_task("Tâche 3")
        assert len(manager.get_tasks()) == 3


class TestTaskManagerFiltering:
    """Tests pour le filtrage des tâches."""
    
    def test_get_tasks_by_status(self, manager_with_tasks):
        todo_tasks = manager_with_tasks.get_tasks(status="todo")
        assert len(todo_tasks) == 4  # Toutes sont "todo" au début
    
    def test_get_tasks_by_owner(self, manager_with_tasks):
        alice_tasks = manager_with_tasks.get_tasks(owner="alice")
        assert len(alice_tasks) == 3  # 3 tâches d'alice (owner par défaut)
    
    def test_get_my_tasks(self, manager_with_tasks):
        """Test du current_user — get_my_tasks filtre sur l'utilisateur courant."""
        my_tasks = manager_with_tasks.get_my_tasks()
        assert len(my_tasks) == 3
        assert all(t.owner == "alice" for t in my_tasks)
    
    def test_get_tasks_combined_filters(self, manager_with_tasks):
        manager_with_tasks.complete_task("Écrire les tests")
        completed_alice = manager_with_tasks.get_tasks(
            status="completed", owner="alice"
        )
        assert len(completed_alice) == 1
    
    def test_get_high_priority_tasks(self, manager_with_tasks):
        urgent = manager_with_tasks.get_high_priority_tasks()
        assert len(urgent) == 2  # "high" et "critical"
        priorities = {t.priority for t in urgent}
        assert priorities == {"high", "critical"}
    
    def test_high_priority_excludes_completed(self, manager_with_tasks):
        manager_with_tasks.complete_task("Écrire les tests")  # was high
        urgent = manager_with_tasks.get_high_priority_tasks()
        assert len(urgent) == 1  # Only "critical" remains


class TestTaskManagerCompletion:
    """Tests pour la complétion de tâches."""
    
    def test_complete_task(self, manager_with_tasks):
        task = manager_with_tasks.complete_task("Écrire les tests")
        assert task.status == "completed"
    
    def test_complete_nonexistent_task_raises(self, manager):
        with pytest.raises(ValueError, match="non trouvée"):
            manager.complete_task("N'existe pas")
    
    def test_count_by_status(self, manager_with_tasks):
        manager_with_tasks.complete_task("Écrire les tests")
        counts = manager_with_tasks.count_by_status()
        assert counts["todo"] == 3
        assert counts["completed"] == 1


class TestTaskManagerParameterized:
    """Tests paramétrés pour couvrir plus de cas."""
    
    @pytest.mark.parametrize("priority", ["low", "medium", "high", "critical"])
    def test_add_task_with_all_valid_priorities(self, manager, priority):
        task = manager.add_task(f"Tâche {priority}", priority=priority)
        assert task.priority == priority
    
    @pytest.mark.parametrize("title, owner, expected_owner", [
        ("Tâche 1", None, "alice"),       # fallback vers current_user
        ("Tâche 2", "bob", "bob"),         # owner explicite
        ("Tâche 3", "charlie", "charlie"), # autre owner explicite
    ])
    def test_owner_assignment(self, manager, title, owner, expected_owner):
        task = manager.add_task(title, owner=owner)
        assert task.owner == expected_owner
```

### 7.3 Points clés de cet exemple

**1. Groupement par classe** — Les tests sont organisés en classes logiques (`TestTask`, `TestTaskManagerCreation`, etc.). C'est propre et navigable.

**2. Fixtures à plusieurs niveaux** — `manager` est vide, `manager_with_tasks` est pré-rempli. On choisit la bonne fixture selon le besoin.

**3. Test du owner fallback** — Quand `owner` n'est pas spécifié, `current_user` est utilisé. Ce pattern est très courant dans les vrais projets.

**4. Tests paramétrés** — On couvre tous les cas de priorité et d'assignation du owner sans duplication.

> 🧠 **À retenir :** Organiser les tests en classes thématiques avec des fixtures ciblées, c'est la marque d'un développeur qui maîtrise pytest.

### 📝 Mini exercice 7

Ajoute ces fonctionnalités à `TaskManager` et écris les tests correspondants :
1. Méthode `delete_task(title)` qui supprime une tâche
2. Méthode `reassign_task(title, new_owner)` qui change le propriétaire
3. Teste les cas normaux ET les cas d'erreur (tâche non trouvée, etc.)

---

## 8. Gestion des erreurs et exceptions

### 8.1 Pourquoi tester les erreurs ?

**Analogie :** Un bon garde du corps ne protège pas seulement contre les attaques connues — il anticipe les situations imprévues. De même, un bon code ne gère pas seulement les cas "heureux" — il gère aussi les erreurs proprement.

Tester les erreurs est **tout aussi important** que tester les cas normaux. Un code professionnel doit échouer de manière prévisible et informative.

### 8.2 `pytest.raises` — L'outil de base

```python
import pytest

def divide(a, b):
    if b == 0:
        raise ValueError("Division par zéro impossible")
    return a / b

# --- Test basique : vérifier qu'une exception est levée ---
def test_divide_by_zero_raises():
    with pytest.raises(ValueError):
        divide(10, 0)

# --- Test avancé : vérifier le message d'erreur ---
def test_divide_by_zero_message():
    with pytest.raises(ValueError, match="Division par zéro"):
        divide(10, 0)

# --- Test : capturer l'exception pour l'inspecter ---
def test_divide_by_zero_inspection():
    with pytest.raises(ValueError) as exc_info:
        divide(10, 0)
    
    assert "zéro" in str(exc_info.value)
    assert exc_info.type == ValueError
```

### 8.3 Le paramètre `match` — Regex puissant

```python
import pytest

class UserService:
    def create_user(self, name, email, age):
        if not name:
            raise ValueError("Le nom est obligatoire")
        if "@" not in email:
            raise ValueError(f"Email invalide : {email}")
        if age < 0:
            raise ValueError(f"Âge invalide : {age}")
        if age < 13:
            raise ValueError("L'utilisateur doit avoir au moins 13 ans")
        return {"name": name, "email": email, "age": age}

@pytest.fixture
def user_service():
    return UserService()

def test_empty_name_raises(user_service):
    with pytest.raises(ValueError, match="nom est obligatoire"):
        user_service.create_user("", "test@mail.com", 25)

def test_invalid_email_raises(user_service):
    with pytest.raises(ValueError, match="Email invalide"):
        user_service.create_user("Alice", "pas-un-email", 25)

def test_negative_age_raises(user_service):
    with pytest.raises(ValueError, match="Âge invalide"):
        user_service.create_user("Alice", "alice@mail.com", -5)

def test_too_young_raises(user_service):
    with pytest.raises(ValueError, match="au moins 13 ans"):
        user_service.create_user("Bob", "bob@mail.com", 10)
```

### 8.4 Tester plusieurs types d'exceptions

```python
import pytest

def process_data(data):
    if data is None:
        raise TypeError("Les données ne peuvent pas être None")
    if not isinstance(data, list):
        raise TypeError(f"Attendu une liste, reçu {type(data).__name__}")
    if len(data) == 0:
        raise ValueError("La liste ne peut pas être vide")
    return sum(data) / len(data)

@pytest.mark.parametrize("input_data, expected_error, match_text", [
    (None, TypeError, "ne peuvent pas être None"),
    ("pas une liste", TypeError, "Attendu une liste"),
    ([], ValueError, "ne peut pas être vide"),
])
def test_process_data_errors(input_data, expected_error, match_text):
    with pytest.raises(expected_error, match=match_text):
        process_data(input_data)
```

### 8.5 Tester qu'une exception n'est PAS levée

```python
def test_valid_division_does_not_raise():
    """On vérifie que la division valide fonctionne sans erreur."""
    # Pas besoin de pytest.raises — si une exception est levée,
    # le test échoue automatiquement
    result = divide(10, 2)
    assert result == 5.0
```

> 🧠 **À retenir :** Utilise `pytest.raises(ExceptionType, match="message")` comme standard. Le `match` est une regex, ce qui te donne beaucoup de flexibilité. Teste TOUJOURS les cas d'erreur, pas seulement les cas heureux.

### 📝 Mini exercice 8

Crée une classe `BankAccount` avec :
- `__init__(self, owner, balance=0)` — refuse un solde négatif
- `deposit(amount)` — refuse un montant <= 0
- `withdraw(amount)` — refuse un montant <= 0 et refuse si solde insuffisant

Écris les tests pour **tous les cas d'erreur** avec `pytest.raises`.

---

## 9. Bonnes pratiques PRO

### 9.1 Isolation des tests

**Règle fondamentale :** Chaque test doit être indépendant. L'ordre d'exécution ne doit jamais affecter le résultat.

```python
# ❌ MAUVAIS — Les tests partagent un état mutable
shared_list = []

def test_add_item():
    shared_list.append("item1")
    assert len(shared_list) == 1

def test_list_is_empty():
    # FAIL si exécuté après test_add_item !
    assert len(shared_list) == 0

# ✅ BON — Chaque test a son propre état via fixture
@pytest.fixture
def fresh_list():
    return []

def test_add_item(fresh_list):
    fresh_list.append("item1")
    assert len(fresh_list) == 1

def test_list_is_empty(fresh_list):
    # Toujours OK — fixture crée une nouvelle liste
    assert len(fresh_list) == 0
```

### 9.2 Nommage expressif

```python
# ❌ MAUVAIS — On ne sait pas ce qui est testé
def test_1():
    ...

def test_user():
    ...

def test_it_works():
    ...

# ✅ BON — Le nom raconte une histoire
def test_new_user_has_default_role_member():
    ...

def test_admin_can_delete_any_user():
    ...

def test_expired_token_raises_authentication_error():
    ...

def test_empty_cart_total_is_zero():
    ...
```

**Pattern recommandé :** `test_<quoi>_<condition/action>_<résultat_attendu>`

### 9.3 Le pattern Arrange-Act-Assert (AAA)

C'est LE pattern standard pour écrire des tests clairs :

```python
def test_applying_discount_reduces_total():
    # ARRANGE — Préparer les données
    cart = ShoppingCart()
    cart.add_item("Laptop", price=1000)
    cart.add_item("Mouse", price=50)
    
    # ACT — Exécuter l'action à tester
    cart.apply_discount(percent=10)
    
    # ASSERT — Vérifier le résultat
    assert cart.total == 945.0  # (1000 + 50) * 0.9
```

**Conseil terrain :** Tu n'as pas besoin d'écrire les commentaires AAA à chaque fois. Mais structure toujours tes tests dans cet ordre mental.

### 9.4 Un comportement par test

```python
# ❌ MAUVAIS — Teste trop de choses à la fois
def test_user():
    user = User("Alice", "alice@mail.com")
    assert user.name == "Alice"
    assert user.email == "alice@mail.com"
    user.deactivate()
    assert user.is_active == False
    user.change_email("new@mail.com")
    assert user.email == "new@mail.com"

# ✅ BON — Un comportement ciblé par test
def test_user_creation_sets_name_and_email():
    user = User("Alice", "alice@mail.com")
    assert user.name == "Alice"
    assert user.email == "alice@mail.com"

def test_deactivate_sets_inactive():
    user = User("Alice", "alice@mail.com")
    user.deactivate()
    assert user.is_active == False

def test_change_email_updates_email():
    user = User("Alice", "alice@mail.com")
    user.change_email("new@mail.com")
    assert user.email == "new@mail.com"
```

### 9.5 Ne pas tester l'implémentation, tester le comportement

```python
# ❌ MAUVAIS — Teste l'implémentation interne
def test_tasks_are_stored_in_list():
    manager = TaskManager("alice")
    manager.add_task("Test")
    assert isinstance(manager._tasks, list)  # Détail d'implémentation !
    assert manager._tasks[0].title == "Test"  # Accès direct au privé !

# ✅ BON — Teste le comportement observable
def test_added_task_is_retrievable():
    manager = TaskManager("alice")
    manager.add_task("Test")
    tasks = manager.get_tasks()
    assert len(tasks) == 1
    assert tasks[0].title == "Test"
```

### 9.6 Récapitulatif des bonnes pratiques

| Pratique | Pourquoi |
|----------|----------|
| Tests isolés | Pas de surprises liées à l'ordre d'exécution |
| Noms descriptifs | Le test sert de documentation |
| Pattern AAA | Code lisible et structuré |
| Un comportement par test | Diagnostic rapide en cas d'échec |
| Tester le comportement, pas l'implémentation | Tests robustes au refactoring |
| Fixtures pour le setup commun | Pas de duplication, code DRY |
| Tests rapides | Retour rapide, exécution fréquente |

---

## 10. Pièges fréquents

### 10.1 Piège n°1 — Oublier que les fixtures sont recréées

```python
@pytest.fixture
def counter():
    return {"count": 0}

def test_increment(counter):
    counter["count"] += 1
    assert counter["count"] == 1

def test_counter_starts_at_zero(counter):
    # ✅ Passe ! La fixture crée un NOUVEAU dict à chaque test
    assert counter["count"] == 0
```

**Le piège :** Croire que la modification dans `test_increment` affecte `test_counter_starts_at_zero`. Non ! Chaque test reçoit une instance fraîche (sauf scope="module" ou "session").

### 10.2 Piège n°2 — Muter un objet partagé dans une fixture

```python
# ❌ DANGEREUX avec scope="module"
@pytest.fixture(scope="module")
def shared_config():
    return {"debug": True, "items": []}

def test_add_item(shared_config):
    shared_config["items"].append("item1")
    assert len(shared_config["items"]) == 1

def test_config_is_clean(shared_config):
    # ❌ FAIL ! La liste contient déjà "item1" car scope="module"
    assert len(shared_config["items"]) == 0
```

**Solution :** Utilise `scope="function"` (défaut) pour les données mutables, ou retourne une copie fraîche.

### 10.3 Piège n°3 — Tests qui dépendent de l'heure ou de la date

```python
from datetime import datetime

# ❌ FRAGILE — Dépend de l'heure exacte
def test_task_created_at():
    task = Task("Test", owner="alice")
    assert task.created_at == datetime.now()  # Peut échouer de quelques millisecondes !

# ✅ ROBUSTE — Vérifie une plage raisonnable
def test_task_created_at_is_recent():
    before = datetime.now()
    task = Task("Test", owner="alice")
    after = datetime.now()
    assert before <= task.created_at <= after
```

### 10.4 Piège n°4 — Oublier le `match` dans `pytest.raises`

```python
# ❌ INCOMPLET — Vérifie juste que ValueError est levée
def test_something():
    with pytest.raises(ValueError):
        do_something_risky()
    # Problème : n'importe quel ValueError fait passer le test !

# ✅ PRÉCIS — Vérifie le message spécifique
def test_something():
    with pytest.raises(ValueError, match="spécifique"):
        do_something_risky()
```

### 10.5 Piège n°5 — Tests qui dépendent de l'ordre des éléments

```python
# ❌ FRAGILE — L'ordre des clés d'un set n'est pas garanti
def test_get_unique_roles():
    roles = get_unique_roles()
    assert roles == ["admin", "member", "viewer"]

# ✅ ROBUSTE — Comparer des ensembles
def test_get_unique_roles():
    roles = get_unique_roles()
    assert set(roles) == {"admin", "member", "viewer"}
```

### 10.6 Piège n°6 — Confondre `is` et `==`

```python
# ❌ PIÈGE SUBTIL
def test_result_is_none():
    result = find_user("inconnu")
    assert result == None  # Fonctionne, mais pas idiomatique

# ✅ CORRECT pour None
def test_result_is_none():
    result = find_user("inconnu")
    assert result is None  # Utilise 'is' pour None, True, False
```

### 10.7 Piège n°7 — Tests trop couplés au code source

```python
# ❌ MAUVAIS — Si on renomme la méthode interne, le test casse
def test_internal_method():
    manager = TaskManager("alice")
    result = manager._calculate_score()  # Méthode privée !
    assert result == 42

# ✅ BON — Tester via l'interface publique
def test_high_score_tasks_appear_first():
    manager = TaskManager("alice")
    manager.add_task("Urgent", priority="critical")
    manager.add_task("Normal", priority="low")
    tasks = manager.get_sorted_tasks()
    assert tasks[0].priority == "critical"
```

> 🧠 **À retenir :** Les pièges les plus courants sont liés à l'isolation (état partagé), la fragilité (dépendance au temps, à l'ordre), et le couplage (tester des détails privés). Garde ces pièges en tête quand tu écris tes tests.

---

## 11. Mini projet guidé — TaskManager

### 11.1 Le projet

On va construire un mini data pipeline testable : un système qui lit des données CSV, les transforme et produit un rapport.

**Structure du projet :**

```
data_pipeline/
├── src/
│   ├── __init__.py
│   ├── reader.py          ← Lecture des données
│   ├── transformer.py     ← Transformation / nettoyage
│   └── reporter.py        ← Génération de rapports
├── tests/
│   ├── __init__.py
│   ├── conftest.py        ← Fixtures partagées
│   ├── test_reader.py
│   ├── test_transformer.py
│   └── test_reporter.py
├── data/
│   └── sample.csv
├── pytest.ini
└── requirements.txt
```

### 11.2 Le code source

**`src/reader.py`** :

```python
"""Module de lecture de données."""
import csv
from pathlib import Path
from typing import Optional


class DataReader:
    """Lit des fichiers CSV et retourne des dictionnaires."""
    
    def __init__(self, filepath: str):
        self.filepath = Path(filepath)
        if not self.filepath.suffix == ".csv":
            raise ValueError(f"Format non supporté : {self.filepath.suffix}")
    
    def read(self) -> list[dict]:
        """Lit le fichier CSV et retourne une liste de dictionnaires."""
        if not self.filepath.exists():
            raise FileNotFoundError(f"Fichier non trouvé : {self.filepath}")
        
        with open(self.filepath, newline="", encoding="utf-8") as f:
            reader = csv.DictReader(f)
            return list(reader)
    
    def read_column(self, column_name: str) -> list[str]:
        """Lit une seule colonne du CSV."""
        data = self.read()
        if not data:
            return []
        if column_name not in data[0]:
            raise KeyError(f"Colonne non trouvée : {column_name}")
        return [row[column_name] for row in data]
```

**`src/transformer.py`** :

```python
"""Module de transformation de données."""
from typing import Optional


class DataTransformer:
    """Transforme et nettoie des données."""
    
    def __init__(self, data: list[dict]):
        if not isinstance(data, list):
            raise TypeError("Les données doivent être une liste")
        self.data = data
        self._original_count = len(data)
    
    @property
    def record_count(self) -> int:
        return len(self.data)
    
    @property
    def dropped_count(self) -> int:
        return self._original_count - len(self.data)
    
    def filter_by(self, column: str, value: str) -> "DataTransformer":
        """Filtre les lignes où column == value."""
        self.data = [row for row in self.data if row.get(column) == value]
        return self  # Permet le chaînage
    
    def exclude_empty(self, column: str) -> "DataTransformer":
        """Supprime les lignes où la colonne est vide."""
        self.data = [
            row for row in self.data
            if row.get(column, "").strip() != ""
        ]
        return self
    
    def add_column(self, name: str, default: str = "") -> "DataTransformer":
        """Ajoute une colonne avec une valeur par défaut."""
        for row in self.data:
            row[name] = default
        return self
    
    def rename_column(self, old_name: str, new_name: str) -> "DataTransformer":
        """Renomme une colonne."""
        for row in self.data:
            if old_name in row:
                row[new_name] = row.pop(old_name)
        return self
    
    def to_list(self) -> list[dict]:
        """Retourne les données transformées."""
        return self.data.copy()
    
    def compute_stats(self, column: str) -> dict:
        """Calcule des statistiques sur une colonne numérique."""
        values = []
        for row in self.data:
            try:
                values.append(float(row[column]))
            except (ValueError, KeyError):
                continue
        
        if not values:
            return {"count": 0, "sum": 0, "mean": 0, "min": 0, "max": 0}
        
        return {
            "count": len(values),
            "sum": round(sum(values), 2),
            "mean": round(sum(values) / len(values), 2),
            "min": min(values),
            "max": max(values),
        }
```

**`src/reporter.py`** :

```python
"""Module de génération de rapports."""
from datetime import datetime


class Reporter:
    """Génère des rapports à partir de données transformées."""
    
    def __init__(self, data: list[dict], title: str = "Rapport"):
        self.data = data
        self.title = title
        self.generated_at = datetime.now()
    
    def summary(self) -> dict:
        """Génère un résumé."""
        if not self.data:
            return {
                "title": self.title,
                "total_records": 0,
                "columns": [],
                "generated_at": self.generated_at.isoformat(),
            }
        
        return {
            "title": self.title,
            "total_records": len(self.data),
            "columns": list(self.data[0].keys()),
            "generated_at": self.generated_at.isoformat(),
        }
    
    def to_markdown(self) -> str:
        """Génère un rapport en Markdown."""
        if not self.data:
            return f"# {self.title}\n\nAucune donnée."
        
        columns = list(self.data[0].keys())
        lines = [
            f"# {self.title}",
            "",
            "| " + " | ".join(columns) + " |",
            "| " + " | ".join(["---"] * len(columns)) + " |",
        ]
        
        for row in self.data:
            values = [str(row.get(col, "")) for col in columns]
            lines.append("| " + " | ".join(values) + " |")
        
        lines.append("")
        lines.append(f"*{len(self.data)} enregistrements*")
        return "\n".join(lines)
```

### 11.3 Les tests complets

**`tests/conftest.py`** :

```python
"""Fixtures partagées pour tous les tests du pipeline."""
import pytest
import os
import tempfile

@pytest.fixture
def sample_csv_data():
    """Données CSV brutes sous forme de liste de dicts."""
    return [
        {"name": "Alice", "department": "Engineering", "salary": "75000"},
        {"name": "Bob", "department": "Marketing", "salary": "65000"},
        {"name": "Charlie", "department": "Engineering", "salary": "80000"},
        {"name": "Diana", "department": "Marketing", "salary": ""},
        {"name": "Eve", "department": "Engineering", "salary": "90000"},
    ]

@pytest.fixture
def sample_csv_file(sample_csv_data):
    """Crée un vrai fichier CSV temporaire."""
    content = "name,department,salary\n"
    for row in sample_csv_data:
        content += f"{row['name']},{row['department']},{row['salary']}\n"
    
    with tempfile.NamedTemporaryFile(
        mode="w", suffix=".csv", delete=False, encoding="utf-8"
    ) as f:
        f.write(content)
        filepath = f.name
    
    yield filepath  # Le test reçoit le chemin du fichier
    
    # Teardown : supprimer le fichier temporaire
    if os.path.exists(filepath):
        os.remove(filepath)

@pytest.fixture
def empty_csv_file():
    """Crée un CSV vide (juste le header)."""
    with tempfile.NamedTemporaryFile(
        mode="w", suffix=".csv", delete=False, encoding="utf-8"
    ) as f:
        f.write("name,department,salary\n")
        filepath = f.name
    
    yield filepath
    
    if os.path.exists(filepath):
        os.remove(filepath)
```

**`tests/test_reader.py`** :

```python
"""Tests pour le module de lecture."""
import pytest
from src.reader import DataReader


class TestDataReaderCreation:
    
    def test_create_with_csv_file(self, sample_csv_file):
        reader = DataReader(sample_csv_file)
        assert reader.filepath.suffix == ".csv"
    
    def test_create_with_non_csv_raises(self):
        with pytest.raises(ValueError, match="Format non supporté"):
            DataReader("data.xlsx")


class TestDataReaderRead:
    
    def test_read_returns_list_of_dicts(self, sample_csv_file):
        reader = DataReader(sample_csv_file)
        data = reader.read()
        assert isinstance(data, list)
        assert all(isinstance(row, dict) for row in data)
    
    def test_read_correct_count(self, sample_csv_file):
        reader = DataReader(sample_csv_file)
        data = reader.read()
        assert len(data) == 5
    
    def test_read_correct_columns(self, sample_csv_file):
        reader = DataReader(sample_csv_file)
        data = reader.read()
        assert set(data[0].keys()) == {"name", "department", "salary"}
    
    def test_read_nonexistent_file_raises(self):
        reader = DataReader("fichier_inexistant.csv")
        with pytest.raises(FileNotFoundError):
            reader.read()
    
    def test_read_empty_csv(self, empty_csv_file):
        reader = DataReader(empty_csv_file)
        data = reader.read()
        assert data == []


class TestDataReaderColumn:
    
    def test_read_column(self, sample_csv_file):
        reader = DataReader(sample_csv_file)
        names = reader.read_column("name")
        assert "Alice" in names
        assert len(names) == 5
    
    def test_read_nonexistent_column_raises(self, sample_csv_file):
        reader = DataReader(sample_csv_file)
        with pytest.raises(KeyError, match="Colonne non trouvée"):
            reader.read_column("inexistant")
```

**`tests/test_transformer.py`** :

```python
"""Tests pour le module de transformation."""
import pytest
from src.transformer import DataTransformer


@pytest.fixture
def transformer(sample_csv_data):
    return DataTransformer(sample_csv_data)


class TestTransformerCreation:
    
    def test_create_with_valid_data(self, sample_csv_data):
        t = DataTransformer(sample_csv_data)
        assert t.record_count == 5
    
    def test_create_with_invalid_data_raises(self):
        with pytest.raises(TypeError):
            DataTransformer("pas une liste")


class TestTransformerFilter:
    
    def test_filter_by_department(self, transformer):
        result = transformer.filter_by("department", "Engineering").to_list()
        assert len(result) == 3
        assert all(r["department"] == "Engineering" for r in result)
    
    def test_filter_no_match(self, transformer):
        result = transformer.filter_by("department", "HR").to_list()
        assert len(result) == 0
    
    def test_exclude_empty_salary(self, transformer):
        result = transformer.exclude_empty("salary").to_list()
        assert len(result) == 4  # Diana avait un salaire vide
    
    def test_chaining_operations(self, transformer):
        result = (
            transformer
            .filter_by("department", "Engineering")
            .exclude_empty("salary")
            .to_list()
        )
        assert len(result) == 3


class TestTransformerColumns:
    
    def test_add_column(self, transformer):
        result = transformer.add_column("status", "active").to_list()
        assert all(r["status"] == "active" for r in result)
    
    def test_rename_column(self, transformer):
        result = transformer.rename_column("name", "full_name").to_list()
        assert "full_name" in result[0]
        assert "name" not in result[0]


class TestTransformerStats:
    
    def test_compute_stats(self, transformer):
        # D'abord exclure les salaires vides
        transformer.exclude_empty("salary")
        stats = transformer.compute_stats("salary")
        assert stats["count"] == 4
        assert stats["mean"] == 77500.0
        assert stats["min"] == 65000.0
        assert stats["max"] == 90000.0
    
    def test_stats_empty_data(self):
        t = DataTransformer([])
        stats = t.compute_stats("salary")
        assert stats["count"] == 0

    def test_dropped_count(self, transformer):
        transformer.filter_by("department", "Engineering")
        assert transformer.dropped_count == 2
```

**`tests/test_reporter.py`** :

```python
"""Tests pour le module de reporting."""
import pytest
from src.reporter import Reporter


@pytest.fixture
def reporter(sample_csv_data):
    return Reporter(sample_csv_data, title="Rapport Test")


class TestReporterSummary:
    
    def test_summary_has_correct_title(self, reporter):
        summary = reporter.summary()
        assert summary["title"] == "Rapport Test"
    
    def test_summary_has_correct_count(self, reporter):
        summary = reporter.summary()
        assert summary["total_records"] == 5
    
    def test_summary_lists_columns(self, reporter):
        summary = reporter.summary()
        assert "name" in summary["columns"]
        assert "department" in summary["columns"]
    
    def test_empty_data_summary(self):
        reporter = Reporter([], title="Vide")
        summary = reporter.summary()
        assert summary["total_records"] == 0
        assert summary["columns"] == []


class TestReporterMarkdown:
    
    def test_markdown_contains_title(self, reporter):
        md = reporter.to_markdown()
        assert "# Rapport Test" in md
    
    def test_markdown_contains_data(self, reporter):
        md = reporter.to_markdown()
        assert "Alice" in md
        assert "Engineering" in md
    
    def test_markdown_has_table_format(self, reporter):
        md = reporter.to_markdown()
        assert "| name |" in md or "| name " in md
        assert "| --- |" in md or "| ---" in md
    
    def test_markdown_shows_record_count(self, reporter):
        md = reporter.to_markdown()
        assert "5 enregistrements" in md
    
    def test_empty_data_markdown(self):
        reporter = Reporter([], title="Vide")
        md = reporter.to_markdown()
        assert "Aucune donnée" in md
```

**`pytest.ini`** :

```ini
[pytest]
testpaths = tests
python_files = test_*.py
python_functions = test_*
addopts = -v --tb=short
```

### 11.4 Exécution

```bash
$ cd data_pipeline
$ pytest

========================= test session starts =========================
collected 30 items

tests/test_reader.py ........                                    [ 27%]
tests/test_transformer.py ...........                            [ 63%]
tests/test_reporter.py .........                                 [100%]

========================= 30 passed in 0.15s ==========================
```

> 🧠 **À retenir :** Ce mini projet montre un pattern professionnel : code modulaire dans `src/`, tests miroirs dans `tests/`, fixtures dans `conftest.py`, et chaque module est testé indépendamment.

---

## 12. Structure d'un projet professionnel

### 12.1 L'arborescence complète

```
mon_projet/
│
├── src/                          ← Code source
│   ├── __init__.py
│   ├── models/                   ← Modèles de données
│   │   ├── __init__.py
│   │   ├── user.py
│   │   └── task.py
│   ├── services/                 ← Logique métier
│   │   ├── __init__.py
│   │   ├── auth_service.py
│   │   └── task_service.py
│   └── utils/                    ← Utilitaires
│       ├── __init__.py
│       ├── validators.py
│       └── formatters.py
│
├── tests/                        ← Tests (miroir de src/)
│   ├── __init__.py
│   ├── conftest.py               ← Fixtures GLOBALES
│   ├── unit/                     ← Tests unitaires
│   │   ├── __init__.py
│   │   ├── conftest.py           ← Fixtures pour tests unitaires
│   │   ├── models/
│   │   │   ├── test_user.py
│   │   │   └── test_task.py
│   │   └── services/
│   │       ├── test_auth_service.py
│   │       └── test_task_service.py
│   └── integration/              ← Tests d'intégration
│       ├── __init__.py
│       ├── conftest.py           ← Fixtures pour tests d'intégration
│       └── test_api.py
│
├── .github/
│   └── workflows/
│       └── tests.yml             ← CI/CD
│
├── pyproject.toml                ← Configuration projet + pytest
├── requirements.txt              ← Dépendances de production
├── requirements-dev.txt          ← Dépendances de développement
├── .gitignore
└── README.md
```

### 12.2 Les fichiers de configuration

**`pyproject.toml`** :

```toml
[project]
name = "mon-projet"
version = "1.0.0"
requires-python = ">=3.10"

[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py"]
python_functions = ["test_*"]
addopts = "-v --tb=short --strict-markers"
markers = [
    "slow: tests lents (intégration, e2e)",
    "integration: tests d'intégration",
]

[tool.coverage.run]
source = ["src"]
omit = ["tests/*"]

[tool.coverage.report]
show_missing = true
fail_under = 80
```

**`requirements-dev.txt`** :

```
pytest>=8.0
pytest-cov>=6.0
pytest-xdist>=3.5      # Tests en parallèle
```

**`.gitignore`** (extraits pertinents) :

```
__pycache__/
*.pyc
.pytest_cache/
htmlcov/
.coverage
```

### 12.3 Le conftest.py multi-niveaux

```python
# tests/conftest.py — Fixtures disponibles PARTOUT
import pytest

@pytest.fixture
def app_config():
    return {"env": "test", "debug": True}

# tests/unit/conftest.py — Fixtures pour les tests UNITAIRES
import pytest

@pytest.fixture
def mock_database():
    """Fausse base de données en mémoire."""
    return FakeDatabase()

# tests/integration/conftest.py — Fixtures pour les tests D'INTÉGRATION
import pytest

@pytest.fixture(scope="module")
def real_database():
    """Vraie connexion à la base de test."""
    db = connect_to_test_db()
    yield db
    db.close()
```

### 12.4 Utilisation des markers

```python
import pytest

# Marquer un test comme lent
@pytest.mark.slow
def test_heavy_computation():
    result = process_large_dataset()
    assert len(result) > 0

# Marquer un test comme test d'intégration
@pytest.mark.integration
def test_api_endpoint():
    response = client.get("/api/users")
    assert response.status_code == 200
```

**Exécution sélective :**

```bash
# Lancer uniquement les tests rapides (exclure les lents)
$ pytest -m "not slow"

# Lancer uniquement les tests d'intégration
$ pytest -m integration
```

---

## 13. Intégration avec les outils modernes

### 13.1 pytest + VS Code

**Configuration** — `.vscode/settings.json` :

```json
{
    "python.testing.pytestArgs": [
        "tests",
        "-v"
    ],
    "python.testing.unittestEnabled": false,
    "python.testing.pytestEnabled": true
}
```

Avec cette config, VS Code affiche un panneau "Testing" dans la barre latérale. Tu peux lancer, debugger et voir les résultats de chaque test directement dans l'éditeur.

**Fonctionnalités VS Code :**
- Icône ▶️ à côté de chaque test pour le lancer individuellement
- Icône 🐛 pour lancer un test en mode debug
- Vue arborescente de tous les tests
- Résultats inline dans le code

### 13.2 pytest + Coverage

La couverture de code mesure quel pourcentage de ton code est exécuté par les tests.

```bash
# Installation
pip install pytest-cov

# Lancer les tests avec couverture
$ pytest --cov=src tests/

---------- coverage: ----------
Name                        Stmts   Miss  Cover
------------------------------------------------
src/__init__.py                 0      0   100%
src/reader.py                  25      2    92%
src/transformer.py             45      3    93%
src/reporter.py                30      1    97%
------------------------------------------------
TOTAL                         100      6    94%

# Générer un rapport HTML détaillé
$ pytest --cov=src --cov-report=html tests/
# Ouvre htmlcov/index.html dans ton navigateur
```

**Le rapport HTML** te montre ligne par ligne ce qui est couvert (vert) et ce qui ne l'est pas (rouge). C'est extrêmement utile pour identifier les cas manqués.

**Conseil terrain :** Vise 80% de couverture minimum. 100% est rarement nécessaire (et parfois contre-productif). Les lignes non couvertes sont souvent des cas limites qui méritent attention.

### 13.3 pytest + GitHub Actions (CI/CD)

Crée le fichier `.github/workflows/tests.yml` :

```yaml
name: Tests Python

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        python-version: ["3.10", "3.11", "3.12"]
    
    steps:
      # 1. Récupérer le code
      - uses: actions/checkout@v4
      
      # 2. Installer Python
      - name: Installer Python ${{ matrix.python-version }}
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      
      # 3. Installer les dépendances
      - name: Installer les dépendances
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
          pip install -r requirements-dev.txt
      
      # 4. Lancer les tests avec couverture
      - name: Lancer les tests
        run: |
          pytest --cov=src --cov-report=xml -v
      
      # 5. (Optionnel) Upload du rapport de couverture
      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          file: ./coverage.xml
```

Ce workflow :
- Se déclenche à chaque push sur `main`/`develop` et à chaque Pull Request
- Teste sur Python 3.10, 3.11 et 3.12
- Installe les dépendances
- Lance les tests avec couverture
- Upload le rapport (optionnel, avec Codecov)

**Conseil terrain :** Configurer la CI dès le début du projet. C'est 10 minutes d'investissement qui te sauvent des heures de debug plus tard.

### 13.4 pytest + tests parallèles

```bash
# Installation
pip install pytest-xdist

# Lancer les tests en parallèle (utilise tous les CPU)
$ pytest -n auto

# Lancer les tests avec 4 workers
$ pytest -n 4
```

Utile quand ta suite de tests devient grande (100+ tests).

---

## 🎓 Récapitulatif final

### Ce que tu sais faire maintenant

| Compétence | Section |
|-----------|---------|
| Comprendre pourquoi les tests sont essentiels | §1 |
| Écrire des tests avec pytest | §2 |
| Utiliser les assertions efficacement | §3 |
| Organiser un projet de tests | §4 |
| Créer et utiliser des fixtures | §5 |
| Paramétriser tes tests | §6 |
| Tester des classes réalistes | §7 |
| Gérer les tests d'exceptions | §8 |
| Appliquer les bonnes pratiques pro | §9 |
| Éviter les pièges courants | §10 |
| Construire un projet testable | §11 |
| Structurer un projet pro | §12 |
| Intégrer pytest avec VS Code, CI/CD, Coverage | §13 |

### Les commandes essentielles

```bash
# Commandes quotidiennes
pytest                          # Lancer tous les tests
pytest -v                       # Mode verbeux
pytest -x                       # Arrêter au premier échec
pytest -k "mot_clé"             # Filtrer par nom
pytest --lf                     # Re-lancer les tests échoués

# Couverture
pytest --cov=src                # Voir la couverture
pytest --cov=src --cov-report=html  # Rapport HTML

# Debug
pytest -s                       # Afficher les print()
pytest -v --tb=long             # Traceback détaillé
pytest --pdb                    # Debugger interactif au premier échec
```

### Checklist avant de pousser ton code

- [ ] Tous les tests passent (`pytest`)
- [ ] Couverture >= 80% (`pytest --cov`)
- [ ] Tests nommés clairement
- [ ] Fixtures utilisées (pas de duplication)
- [ ] Cas d'erreur testés (`pytest.raises`)
- [ ] Tests isolés (pas d'état partagé)

> 🧠 **Le mot de la fin :** Les tests ne sont pas un luxe, c'est un investissement. Chaque minute passée à écrire des tests est une heure gagnée en debug. Les meilleurs développeurs ne sont pas ceux qui écrivent le code le plus rapide — ce sont ceux qui écrivent le code le plus fiable. Et ça commence par les tests.

---

*Cours créé pour des développeurs qui visent l'excellence. Bonne pratique !* 🚀
