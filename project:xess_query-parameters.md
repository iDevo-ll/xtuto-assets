# Mini-Projet : Bibliothèque API — Query Parameters & XESS

## C'est quoi les Query Parameters ?

Quand un client envoie une requête HTTP, il peut transmettre des informations supplémentaires directement dans l'URL, après le point d'interrogation `?`. Ces informations s'appellent des **query parameters** (ou paramètres de requête).

```
GET /books?type=romans&page=2
         ^^^^^^^^^^^^^^^^
         query parameters
```

Chaque paramètre est un couple `clé=valeur`, séparé par `&` quand il y en a plusieurs. Ils sont **facultatifs** par nature — c'est au serveur de décider comment il réagit en leur présence ou absence. C'est leur grande différence avec les **path parameters** (ex: `/books/:id`) qui, eux, font partie intégrante de la route.

Ils servent généralement à : **filtrer**, **paginer**, **trier**, ou **rechercher** des ressources sans créer de nouvelles routes.

---

## Objectif du mini-projet

Construire une **API de gestion de livres** avec XyPriss qui met en pratique :

1. **La lecture et validation stricte des query parameters**
2. **XESS** — XyPriss Environment Security & Shield — pour une gestion de session sécurisée basée sur les variables d'environnement

---

## Algorithme général

Le client appelle `GET /books`. Le comportement change selon la présence ou l'absence de queries :

| Requête | Comportement |
|---|---|
| `GET /books` | Retourne tous les livres |
| `GET /books?type=romans` | Filtre par type |
| `GET /books?type=romans&page=2` | Filtre + pagination |
| Query invalide | Erreur `400 Bad Request` |
| Session bloquée | Erreur `401 Unauthorized` |

---

## Queries supportées

| Paramètre | Type | Valeurs acceptées | Défaut |
|---|---|---|---|
| `type` | `string` | `romans` \| `all` | `all` |
| `page` | `number` | Entier positif ≥ 1 | `1` |

---

## Règles de validation (strictes)

- `type` doit être exactement `"romans"` ou `"all"` — toute autre valeur → `400`
- `page` doit être un entier positif — une chaîne, un flottant, un négatif → `400`
- Tout query parameter **non reconnu** est ignoré silencieusement (pas d'erreur)

---

## XESS — XyPriss Environment Security & Shield

### Principe

XESS est le système de XyPriss qui permet de **lire et écrire les variables d'environnement de manière sécurisée et dynamique** au runtime, sans jamais passer par `process.env` directement (qui est masqué par le Shield).

Dans ce projet, on utilise XESS pour implémenter un **mécanisme de blocage de session** côté serveur.

### Comportement attendu

```
Requête valide          → 200 OK
Requête invalide (1x)   → 400 Bad Request  |  ATTEMPT = 1
Requête invalide (2x)   → 400 Bad Request  |  ATTEMPT = 2
Requête invalide (3x)   → 400 Bad Request  |  ATTEMPT = 3  → session BLOQUÉE
Toute requête suivante  → 401 Unauthorized  (peu importe si valide ou non)
```

### Pourquoi stocker dans XESS ?

La variable `ATTEMPT` n'est pas une simple variable locale — elle persiste entre les requêtes grâce à XESS qui la maintient dans l'environnement sécurisé du processus. C'est ce qui rend le comportement **stateful** sans base de données externe.

### Lecture / écriture XESS

```typescript
// Lire le nombre de tentatives
const attempts = __sys__.__env__.get("ATTEMPT") ?? "0";

// Incrémenter et sauvegarder
__sys__.__env__.set("ATTEMPT", String(Number(attempts) + 1));
```

---

## Structure du projet

```
books-api/
├── src/
│   ├── index.ts           # Point d'entrée, création du serveur
│   ├── routes/
│   │   └── books.route.ts # Route GET /books
│   ├── middlewares/
│   │   └── xess.guard.ts  # Middleware de blocage XESS
│   └── data/
│       └── books.ts       # Données mockées
├── .env
└── xypriss.config.jsonc
```

---

## Points pédagogiques clés pour la vidéo

1. **Démontrer** qu'un query absent ≠ une erreur → comportement par défaut propre
2. **Montrer en live** la différence entre un `400` et un `401` dans Postman/Insomnia
3. **Tracer** la variable `ATTEMPT` dans les logs à chaque requête pour visualiser XESS en action
4. **Insister** sur le fait que `process.env.ATTEMPT` retournerait `undefined` (masqué par le Shield) — seul `__sys__.__env__.get()` fonctionne

> ⚠️ **Avertissement pédagogique :** Stocker un compteur de tentatives dans une variable d'environnement runtime est une approche **éducative uniquement**. En production, ce genre de logique doit être géré avec XEMS (sessions chiffrées) ou une base de données dédiée. L'objectif ici est uniquement de comprendre le fonctionnement de XESS.

