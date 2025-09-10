# Refactoring SOLID — Product Manager

Ce document contient le code refactorisé (structure de dépôt), un README expliquant les violations SOLID identifiées, et les fichiers Python correspondants.

---

## Arborescence proposée

```
product_manager/
├─ __init__.py
├─ models.py
├─ repository.py
├─ interfaces.py
├─ validators.py
├─ services/
│  ├─ __init__.py
│  ├─ creator.py
│  ├─ retriever.py
│  ├─ updater.py
│  ├─ deleter.py
│  ├─ searcher.py
│  └─ facade.py
tests/
├─ test_product_manager.py
README.md
```

---

## README.md

```
# TP — Refactoring SOLID d'un Système de Gestion de Produits

## Résumé
Refactorisation d'un petit système de gestion de produits afin de respecter les principes SOLID. Ajout d'une fonctionnalité : recherche par catégorie et prix maximum.

## Violations SOLID identifiées (code initial)

- **SRP (Single Responsibility Principle)**: `ProductService` gère trop de responsabilités — validation, logique CRUD et persistance.
- **OCP (Open/Closed Principle)**: `ProductService` est difficile à étendre (nouvelle validation ou nouvelle source de données demande de modifier la classe).
- **DIP (Dependency Inversion Principle)**: `ProductService` instancie directement `ProductRepository` — dépendance vers une implémentation concrète.
- **ISP (Interface Segregation Principle)**: `ProductRepository` expose une API unique. Après refactor, on découpe en interfaces lecteurs/écrivains si nécessaire.
- **LSP (Liskov Substitution Principle)**: Pas de hiérarchie compliquée dans l'existant, mais le refactor s'assure que les abstractions peuvent être substituées sans surprise.

## Principales modifications

1. **Séparation des responsabilités**
   - Création de classes spécialisées: `ProductCreator`, `ProductRetriever`, `ProductUpdater`, `ProductDeleter`, `ProductSearcher`.
   - `ProductValidator` pour la logique de validation.

2. **AbstractionsP**
   - Introduction d'interfaces `IProductReader` et `IProductWriter` (regroupées aussi dans `IProductRepository` si besoin).
   - Les services dépendent d'abstractions (interfaces) injectées via constructeur.

3. **ISP**
   - Les interfaces sont fines : lecture/écriture séparées si un client n'a besoin que d'une partie.

4. **OCP**
   - Ajout de nouvelles règles ou de nouvelles sources (ex: base de données, CSV) possible en ajoutant une nouvelle implémentation de `IProductReader` / `IProductWriter` sans toucher aux services.

5. **Nouvelle fonctionnalité**
   - `ProductSearcher` implémente `search_by_category_and_max_price(category: str, max_price: float) -> List[Product]`.

## Exécution & tests

- Installer (si nécessaire): Python 3.9+
- Lancer les tests unitaires: `pytest -q`

## Notes

- Le code est volontairement simple, axé sur la lisibilité et la clarté de l'architecture.
```

---

## Fichiers Python

### product\_manager/models.py

```python
from dataclasses import dataclass

@dataclass
class Product:
    id: int | None = None
    name: str | None = None
    description: str | None = None
    price: float | None = None
    category: str | None = None
```

### product\_manager/interfaces.py

```python
from __future__ import annotations
from typing import Protocol, List, Optional
from product_manager.models import Product

class IProductReader(Protocol):
    def get_by_id(self, product_id: int) -> Optional[Product]:
        ...

    def get_all(self) -> List[Product]:
        ...

class IProductWriter(Protocol):
    def add(self, product: Product) -> Product:
        ...

    def update(self, product: Product) -> bool:
        ...

    def delete(self, product_id: int) -> bool:
        ...


class IProductRepository(IProductReader, IProductWriter, Protocol):
    pass
```

### product\_manager/repository.py

```python
from typing import Dict, List, Optional
from product_manager.models import Product
from product_manager.interfaces import IProductRepository

class ProductRepository(IProductRepository):
   
    def __init__(self):
        self._products: Dict[int, Product] = {}
        self._next_id = 1

    def add(self, product: Product) -> Product:
        product.id = self._next_id
        self._products[self._next_id] = product
        self._next_id += 1
        return product

    def get_by_id(self, product_id: int) -> Optional[Product]:
        return self._products.get(product_id)

    def get_all(self) -> List[Product]:
        return list(self._products.values())

    def update(self, product: Product) -> bool:
        if product.id in self._products:
            self._products[product.id] = product
            return True
        return False

    def delete(self, product_id: int) -> bool:
        if product_id in self._products:
            del self._products[product_id]
            return True
        return False
```

### product\_manager/validators.py

```python
from abc import ABC, abstractmethod
from product_manager.models import Product

class IProductValidator(ABC):
    @abstractmethod
    def validate(self, product: Product) -> None:
        """Lève une exception si invalide."""
        pass

class DefaultProductValidator(IProductValidator):
    def validate(self, product: Product) -> None:
        if not product.name or product.name.strip() == "":
            raise ValueError("Le nom est requis.")
        if product.price is None or product.price <= 0:
            raise ValueError("Le prix doit être un nombre positif.")
```

### product\_manager/services/creator.py

```python
from product_manager.interfaces import IProductWriter
from product_manager.models import Product
from product_manager.validators import IProductValidator

class ProductCreator:
    def __init__(self, writer: IProductWriter, validator: IProductValidator):
        self._writer = writer
        self._validator = validator

    def create(self, name: str, description: str, price: float, category: str) -> Product:
        product = Product(name=name, description=description, price=price, category=category)
        self._validator.validate(product)
        return self._writer.add(product)
```

### product\_manager/services/retriever.py

```python
from typing import List, Optional
from product_manager.interfaces import IProductReader
from product_manager.models import Product

class ProductRetriever:
    def __init__(self, reader: IProductReader):
        self._reader = reader

    def get_by_id(self, product_id: int) -> Optional[Product]:
        return self._reader.get_by_id(product_id)

    def get_all(self) -> List[Product]:
        return self._reader.get_all()
```

### product\_manager/services/updater.py

```python
from product_manager.interfaces import IProductReader, IProductWriter
from product_manager.validators import IProductValidator
from product_manager.models import Product

class ProductUpdater:
    def __init__(self, reader: IProductReader, writer: IProductWriter, validator: IProductValidator):
        self._reader = reader
        self._writer = writer
        self._validator = validator

    def update(self, product_id: int, name: str, description: str, price: float, category: str) -> Product:
        product = self._reader.get_by_id(product_id)
        if not product:
            raise ValueError("Produit non trouvé.")
        product.name = name
        product.description = description
        product.price = price
        product.category = category
        self._validator.validate(product)
        success = self._writer.update(product)
        if not success:
            raise RuntimeError("Échec de la mise à jour.")
        return product
```

### product\_manager/services/deleter.py

```python
from product_manager.interfaces import IProductWriter

class ProductDeleter:
    def __init__(self, writer: IProductWriter):
        self._writer = writer

    def delete(self, product_id: int) -> bool:
        return self._writer.delete(product_id)
```

### product\_manager/services/searcher.py

```python
from typing import List
from product_manager.interfaces import IProductReader
from product_manager.models import Product

class ProductSearcher:
    def __init__(self, reader: IProductReader):
        self._reader = reader

    def search_by_category_and_max_price(self, category: str | None, max_price: float | None) -> List[Product]:
        """Retourne les produits correspondant à la catégorie (si fournie) et au prix <= max_price (si fourni)."""
        products = self._reader.get_all()
        result = []
        for p in products:
            if category and p.category != category:
                continue
            if max_price is not None and p.price is not None and p.price > max_price:
                continue
            result.append(p)
        return result
```

### product\_manager/services/facade.py

```python
from product_manager.interfaces import IProductRepository
from product_manager.validators import DefaultProductValidator
from product_manager.services.creator import ProductCreator
from product_manager.services.retriever import ProductRetriever
from product_manager.services.updater import ProductUpdater
from product_manager.services.deleter import ProductDeleter
from product_manager.services.searcher import ProductSearcher

class ProductServiceFacade:
    
    def __init__(self, repository: IProductRepository):
        validator = DefaultProductValidator()
        self.creator = ProductCreator(repository, validator)
        self.retriever = ProductRetriever(repository)
        self.updater = ProductUpdater(repository, repository, validator)
        self.deleter = ProductDeleter(repository)
        self.searcher = ProductSearcher(repository)
```

### tests/test\_product\_manager.py

```python
import pytest
from product_manager.repository import ProductRepository
from product_manager.services.facade import ProductServiceFacade

@pytest.fixture
def service():
    repo = ProductRepository()
    return ProductServiceFacade(repo)

def test_create_and_get(service):
    p = service.creator.create("Laptop", "Puissant", 1200.0, "Électronique")
    assert p.id is not None
    got = service.retriever.get_by_id(p.id)
    assert got is not None
    assert got.name == "Laptop"

def test_update(service):
    p = service.creator.create("Ecran", "Elegant", 80.0, "Électronique")
    updated = service.updater.update(p.id, "Ecran HD", "Très élégant", 90.0, "Électronique")
    assert updated.name == "Ecran HD"

def test_delete(service):
    p = service.creator.create("Clavier", "Mécanique", 50.0, "Accessoires")
    assert service.deleter.delete(p.id) is True
    assert service.retriever.get_by_id(p.id) is None

def test_search(service):
    service.creator.create("A", "a", 10.0, "C1")
    service.creator.create("B", "b", 20.0, "C1")
    service.creator.create("C", "c", 5.0, "C2")
    res = service.searcher.search_by_category_and_max_price("C1", 15.0)
    assert len(res) == 1
    assert res[0].name == "A"
```

---

## Remarques finales

* Le code montre comment appliquer les principes SOLID en pratique. Les interfaces sont simples et peuvent être étendues.
* Pour ajouter une nouvelle source de données (CSV, DB), il suffit d'implémenter `IProductReader`/`IProductWriter` et d'injecter l'implémentation dans la `ProductServiceFacade`.

---
