# ☕ PayeTonKawa - Architecture Micro-services

Projet de modernisation de l'infrastructure PayeTonKawa.

Passage d'une architecture legacy vers des micro-services conteneurisés avec Node.js (NestJS) et RabbitMQ. Les bases de données sont externalisées et managées via Railway.

## 🏗️ Architecture Hybride

- **Services & Broker :** Hébergés localement via Docker containers.
- **Persistance :** Bases de données PostgreSQL hébergées sur le Cloud (Railway).

### Schéma de l'architecture
```mermaid
graph TD
    User((Utilisateur / Front))

    subgraph "Local : Infrastructure Docker"
        AuthAPI[API Authentification<br/>Port: 3001]
        ClientAPI[API Clients<br/>Port: 3002]
        ProductAPI[API Produits<br/>Port: 3003]
        OrderAPI[API Commandes<br/>Port: 3004]

        RabbitMQ{RabbitMQ<br/>Broker Local}
    end

    subgraph "Cloud : Railway (Bases de données)"
        DB_Auth[(Postgres Auth)]
        DB_Client[(Postgres Clients)]
        DB_Product[(Postgres Produits)]
        DB_Order[(Postgres Commandes)]
    end

    %% Flux Utilisateur
    User -->|Login / JWT| AuthAPI
    User -->|CRUD Client| ClientAPI
    User -->|Consultation Stock| ProductAPI
    User -->|Création Commande| OrderAPI

    %% Connexion aux BDD Distantes
    AuthAPI ---|Internet| DB_Auth
    ClientAPI ---|Internet| DB_Client
    ProductAPI ---|Internet| DB_Product
    OrderAPI ---|Internet| DB_Order

    %% Communication Inter-services
    OrderAPI -.->|HTTP: Vérif Token| AuthAPI
    OrderAPI -.->|HTTP: Vérif Client| ClientAPI

    %% Communication Asynchrone
    OrderAPI --"Event: order_created"--> RabbitMQ
    RabbitMQ --"Update Stock"--> ProductAPI
```

### Tableau des Services

| Service   | Port Local | Description                                | Documentation (Swagger)   |
| --------- | ---------- | ------------------------------------------ | ------------------------- |
| Auth      | 3001       | Gestion des utilisateurs & Tokens JWT      | http://localhost:3001/api |
| Client    | 3002       | CRM Clients                                | http://localhost:3002/api |
| Produits  | 3003       | Catalogue & Gestion des stocks             | http://localhost:3003/api |
| Commandes | 3004       | Prise de commande & Événements asynchrones | http://localhost:3004/api |

### 🛠️ Outils de Monitoring & Administration

| Outil           | Port  | Identifiants par défaut | Description                      |
| :-------------- | :---- | :---------------------- | :------------------------------- |
| **Grafana**     | 3005  | admin / admin           | Tableaux de bord de surveillance |
| **Prometheus**  | 9090  | (Aucun)                 | Collecte des métriques           |
| **RabbitMQ UI** | 15672 | admin / admin123        | Gestion des files d'attente      |

## 🚀 Démarrage Rapide

### Pré-requis

- [Docker & Docker Compose](https://www.docker.com/)
- Une connexion Internet (pour accéder aux BDD Railway)
- Le fichier .env contenant les identifiants Railway (DATABASE_URL)

### Installation & Lancement

1\. Configuration

Assurez-vous que les fichiers .env de chaque service contiennent les bonnes URLs de connexion Railway.

Exemple : postgresql://postgres:password@roundhouse.proxy.rlwy.net:PORT/railway

_Si c'est pas correcte envoyer un mail à stephane.hilaricus@gmail.com_

**2\. Lancer l'infrastructure applicative**

```bash
docker-compose up -d --build
```

> **Note :** Cette commande lance les 4 APIs Node.js, le Broker RabbitMQ ainsi que la stack de monitoring (Prometheus/Grafana). Les bases de données sont déjà actives sur le Cloud.

**3\. Accéder aux interfaces Swagger**

- **Auth :** http://localhost:3001/api
- **Clients :** http://localhost:3002/api
- **Produits :** http://localhost:3003/api
- **Commandes :** http://localhost:3004/api
- **Grafana :** http://localhost:3005
- **RabbitMQ :** UI http://localhost:15672

## ✅ Qualité & Tests (CI/CD)

Chaque service dispose d'un pipeline CI/CD complet (**GitHub Actions**) incluant :

- Linting & Formatage
- Tests Unitaires (**\> 90% de couverture**)
- Audit de sécurité (TruffleHog, CodeQL)
- Vérification du Build Docker

Pour lancer les tests d'un service spécifique en local :

```bash
 cd ../ptk-auth-service
npm install  npm run test:cov
```

## 🔄 Flux de données (RabbitMQ)

Lorsqu'une commande est créée (POST /orders) :

1.  L'API **Commande** vérifie le token (via API Auth) et le client (via API Client).
2.  La commande est sauvegardée dans **Postgres (Railway)**.
3.  Un événement order_created est publié dans **RabbitMQ (Local)**.
4.  Le service **Produits** consomme cet événement et décrémente le stock dans sa propre base Postgres (Railway).

_Projet réalisé dans le cadre du MSPR._
