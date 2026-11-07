# 🚴 Cube Bikes — Plateforme E-Commerce (SAE 3.01)

[![Laravel](https://img.shields.io/badge/Laravel-10.x-FF2D20?style=for-the-badge&logo=laravel&logoColor=white)](https://laravel.com)
[![PHP](https://img.shields.io/badge/PHP-8.1+-777BB4?style=for-the-badge&logo=php&logoColor=white)](https://php.net)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-316192?style=for-the-badge&logo=postgresql&logoColor=white)](https://www.postgresql.org/)
[![TailwindCSS](https://img.shields.io/badge/Tailwind_CSS-v4-38B2AC?style=for-the-badge&logo=tailwind-css&logoColor=white)](https://tailwindcss.com/)
[![Alpine.js](https://img.shields.io/badge/Alpine.js-8BC0D0?style=for-the-badge&logo=alpinedotjs&logoColor=white)](https://alpinejs.dev/)
[![Stripe](https://img.shields.io/badge/Stripe-Cashier-635BFF?style=for-the-badge&logo=stripe&logoColor=white)](https://stripe.com)
[![Gemini AI](https://img.shields.io/badge/Google_Gemini-2.0_Flash-4285F4?style=for-the-badge&logo=google&logoColor=white)](https://deepmind.google/technologies/gemini/)

Plateforme e-commerce complète développée dans le cadre de la **SAE 3.01** (BUT Informatique). Le projet simule le site officiel de vente en ligne de la marque **Cube Bikes**, articulé autour d'une **base de données relationnelle modélisée sur-mesure (MCD/MLD/MPD)** et d'une **architecture applicative robuste en couches** (Services, DTOs, Pipeline de filtres, Assistant IA contextuel, Multi-Guard, RGPD).

---

## 📌 Sommaire

1. [Modélisation & Conception de la Base de Données](#-modélisation--conception-de-la-base-de-données)
2. [Architecture Applicative](#-architecture-applicative)
3. [Assistant IA Contextuel (Gemini 2.0 Flash)](#-assistant-ia-contextuel-gemini-20-flash)
4. [Sécurité & Conformité RGPD](#-sécurité--conformité-rgpd)
5. [Fonctionnalités Clés](#-fonctionnalités-clés)
6. [Stack Technique](#-stack-technique)
7. [Installation & Prise en Main](#-installation--prise-en-main)
8. [Équipe & Contributeurs](#-équipe--contributeurs)

---

## 🗄️ Modélisation & Conception de la Base de Données

La base de données PostgreSQL a été **intégralement pensée et modélisée par l'équipe**, depuis l'analyse du domaine métier jusqu'au déploiement du schéma physique :

```
[ Besoins Métier ] ──► [ MCD : Entités / Associations ] ──► [ MLD : Normalisation 3NF ] ──► [ MPD : Schéma PostgreSQL ]
```

### 1. Décisions de modélisation (MCD / MLD)
* **Héritage & Spécialisation des données :**
  * `Article` constitue l'entité parente générique (prix, nom, visuels, catégorie).
  * Spécialisation par sous-typage vers `Velo` (géométrie, matériaux, millésime) et `Accessoire`.
  * Spécialisation ultérieure de `Velo` vers `VeloElectrique` (moteur, capacité batterie, autonomie).
* **Déclinaisons & Matrice de variantes :**
  * Découplage entre l'article conceptuel et ses déclinaisons physiques via les entités `ArticleReference` / `BikeReference` (combinaisons couleur, taille de cadre, capacité de batterie).
* **Stocks multi-magasins :**
  * Entité d'association `ShopAvailability` liant référence, magasin et taille, permettant le suivi granulaire des statuts de stock (*en stock*, *commandable*, *épuisé*).
* **Relations n-aires et réflexives :**
  * Compatibilité certifiée accessoires/vélos (`compatibleAccessories`).
  * Recommandations d'articles similaires (`similarArticles`).
* **Traçabilité temporelle des commandes :**
  * Historisation des états de commande via une table pivot horodatée (`historique_etat`) reliant `Order` et `OrderState`.

### 2. Contraintes d'intégrité SQL (MPD)
* **Intégrité référentielle stricte :** Clés étrangères (`FOREIGN KEY`) avec clauses d'action adaptées (`ON DELETE RESTRICT` pour protéger les factures, `ON DELETE CASCADE` pour les données dépendantes).
* **Contraintes de domaine & d'unicité :** Contraintes `CHECK` sur les montants positifs et les dates, contraintes `UNIQUE` sur les références articles, emails et codes promo.
* **Typage adapté :** Utilisation des types natifs PostgreSQL (`numeric`, `timestamp`, `boolean`, `varchar`) garantissant cohérence et performance.

---

## 🏗️ Architecture Applicative

Le projet applique les principes de la **Clean Architecture** et du découplage **SOLID** en évitant les contrôleurs surchargés ("Fat Controllers").

```
                                  FLUX DE TRAITEMENT DES REQUÊTES
                                  
  Navigateur Client
         │
         ▼
  [ Routes (web.php) ] ──► [ Middleware (Auth / Role Guard) ]
                                    │
                                    ▼
                         [ FormRequest (Validation) ]
                                    │
                                    ▼
                         [ Controller (Orchestration) ]
                                    │
                    ┌───────────────┴───────────────┐
                    ▼                               ▼
          [ Services Métier ]             [ Pipeline de Filtres ]
       (Cart, Order, Gdpr, Assistant)    (FilterEngineService + Filters)
                    │                               │
                    └───────────────┬───────────────┘
                                    ▼
                         [ DTOs (Données Typées) ]
                                    │
                                    ▼
                      [ Modèles Eloquent & Scopes ]
                                    │
                                    ▼
                          [ PostgreSQL (MPD) ]
```

### 1. Découpage en couches

#### A. Contrôleurs fins (Slim Controllers)
Les contrôleurs ne contiennent aucune logique métier : ils reçoivent la requête validée par une `FormRequest`, appellent le ou les services appropriés, et injectent les DTOs dans les vues.

#### B. Couche Service (Domain Services)
Toute la logique métier réside dans des services dédiés et testables :
* `CartService` & `CheckoutService` : Calcul des sous-totaux, remises, frais de port conditionnels, orchestration des transactions de commande (`DB::transaction`).
* `FilterEngineService` : Moteur de recherche et de facettes.
* `CubeAssistantService` : Construction du prompt contextuel et échange avec l'IA.
* `GdprService` : Gestion de l'anonymisation légale et des exports de données.
* `TwoFactorService` : Chiffrement des secrets TOTP et validation des codes 2FA.

#### C. Objets de Transfert de Données (DTOs)
Pour éliminer les tableaux associatifs non typés ("array typing blindness"), des **DTOs immuables** (exploitant le *Constructor Property Promotion* de PHP 8) assurent les échanges entre les services, les contrôleurs et les vues :
* `OrderSummaryDTO`, `OrderFinancialsDTO`, `CheckoutDataDTO`
* `CartItemDTO`, `CartViewDataDTO`, `ShippingModeDTO`
* `VariantOptionDTO`, `SizeOptionDTO`, `ArticlePriceDTO`

#### D. Moteur de Filtres (Pipeline / Specification Pattern)
Le filtrage du catalogue ne repose pas sur une accumulation de conditions `if` imbriquées. Chaque critère est encapsulé dans une classe autonome implémentant l'interface `ArticleFilter` :
* `PriceFilter`, `CategoryFilter`, `ColorFilter`, `SizeFilter`, `FrameFilter`, `MaterialFilter`, `VintageFilter`, `AvailabilityFilter`...
* `FilterEngineService` compose et applique dynamiquement ce pipeline sur le Query Builder Eloquent en fonction des paramètres de recherche.

#### E. Adaptation Eloquent ◄► MPD (`CustomUserProvider`)
Comme le schéma SQL conçu par l'équipe utilise des clés primaires et des colonnes explicites en français (`id_client`, `email_client`, `hash_mdp_client`), un **`CustomUserProvider`** a été développé pour faire le pont avec le système d'authentification natif de Laravel sans dénaturer le schéma de la base de données.

---

## 🤖 Assistant IA Contextuel (Gemini 2.0 Flash)

L'assistant intelligent intégré repose sur une **architecture contextuelle et situationnelle** :

```
Page consultée par l'utilisateur (ex: VTT Stereo 140)
  │
  ├─► Extraction de l'état (ID article, variantes affichées, panier...)
  │
  ▼
[ CubeAssistantService ]
  ├─► 1. Charge le Prompt Système statique (règles métier, ton, limites d'action)
  ├─► 2. Sérialise le Payload Situationnel (specs réelles, stock, prix, promos)
  └─► 3. Formate la requête vers l'API Gemini 2.0 Flash
        │
        ▼
  [ Réponse markdown générée ] ──► [ Affichage widget BotMan ]
```

* **Séparation stricte Données Froides / Données Chaudes :**
  * **Données Froides (déterministes) :** Prix, stocks, tailles disponibles proviennent **strictement du payload JSON de la BDD**. L'IA a interdiction stricte d'inventer des disponibilités.
  * **Données Chaudes (culture cycliste) :** L'IA est autorisée à exploiter ses connaissances pour expliquer les termes techniques (ex: différences entre carbone HPC et aluminium HPA, transmission Shimano XT vs Deore).

---

## 🛡️ Sécurité & Conformité RGPD

* **Multi-Guard & RBAC (Role-Based Access Control) :**
  * Guard `web` pour les clients (`Client`).
  * Guard `staff` pour les collaborateurs internes (`StaffUser`), partitionné par rôles : `COMMERCIAL`, `DIRECTOR` (accès aux stats), et `DPO`.
* **Authentification Double Facteur (2FA TOTP) :**
  * Implémentation via `PragmaRX/Google2FA`.
  * Clés secrètes et codes de secours **chiffrés au repos** dans PostgreSQL.
  * Confirmation de mot de passe obligatoire pour toute action critique (désactivation 2FA, régénération des codes).
* **Conformité RGPD "by design" :**
  * **Droit à l'oubli articulé avec les obligations comptables :** Anonymisation irréversible des clients ayant commandé (conservation de la facture avec masquage des données personnelles) ou suppression totale si aucun historique d'achat.
  * **Espace DPO :** Outil de purge automatique des comptes inactifs depuis plus de 3 ans.
  * **Audience & Traceurs :** Gestionnaire de consentement *Tarteaucitron* et tracking privacy-friendly via *Matomo*.

---

## 🌟 Fonctionnalités Clés

* **Boutique & Catalogue :**
  * Moteur de recherche plein texte PostgreSQL (`ILIKE`).
  * **Visualiseur 360° interactif** sur les fiches vélos (composant Alpine.js avec gestion du glisser/déposer).
  * Affichage automatique des accessoires compatibles validés par Cube.
* **Store Locator & Disponibilités :**
  * Calcul de distance géographique en temps réel (géolocalisation HTML5).
  * Modal interactive avec vue liste et vue carte des boutiques Cube agréées.
* **Panier & Tunnel d'achat :**
  * Gestion de codes promotionnels uniques.
  * Règles de livraison contextuelles (ex: retrait atelier obligatoire pour les vélos avec montage pro).
  * Paiement en ligne via **Stripe Checkout** (Laravel Cashier).
* **Dashboard Client :**
  * Gestion d'adresses multiples avec autocomplétion **Google Places API**.
  * Historique détaillé des commandes et téléchargement de factures.
  * Export complet des données personnelles au format JSON (portabilité RGPD).

---

## 💻 Stack Technique

| Couche | Technologies |
| :--- | :--- |
| **Backend** | PHP 8.1+, Laravel 10.x |
| **Base de Données** | PostgreSQL (Schéma sur-mesure MCD/MLD/MPD) |
| **Architecture** | Service-Layer, DTOs, Pipeline/Specification Filter Pattern |
| **Frontend** | Blade Components, Tailwind CSS v4, Alpine.js, Vite |
| **Intelligence Artificielle**| Google Gemini API (`gemini-2.0-flash`), BotMan Studio |
| **Paiement & APIs** | Stripe (Cashier), Google Places API, Google OAuth (Socialite) |
| **Sécurité** | 2FA / TOTP (Google Authenticator), Multi-Guards, CSRF/XSS protection |
| **Conformité & Privacy** | Tarteaucitron.js, Matomo Analytics |
| **Qualité & Tooling** | Laravel Pint, PHPUnit, Prettier, Python PIL (redimensionnement d'images) |

---

## ⚙️ Installation & Prise en Main

### 1. Prérequis
* PHP >= 8.1 avec extensions : `pdo_pgsql`, `gd`, `curl`, `mbstring`
* Composer >= 2.x
* Node.js >= 18.x & NPM
* Serveur PostgreSQL actif

### 2. Cloner et installer les dépendances
```bash
git clone https://github.com/ForWarZz/sae3.01-cube.git
cd sae3.01-cube

# Dépendances PHP
composer install

# Dépendances Node.js
npm install
```

### 3. Configuration de l'environnement
```bash
cp .env.exemple .env
php artisan key:generate
```

Éditer le fichier `.env` avec vos identifiants PostgreSQL et clés d'API :
```ini
APP_NAME=CubeBikes
APP_URL=http://localhost:8000

# Base de données PostgreSQL (MPD Cube)
DB_CONNECTION=pgsql
DB_HOST=127.0.0.1
DB_PORT=5432
DB_DATABASE=cube_bikes
DB_USERNAME=votre_utilisateur
DB_PASSWORD=votre_mot_de_passe

# Stripe (Paiements)
STRIPE_KEY=pk_test_...
STRIPE_SECRET=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...

# Google APIs (Places & Socialite)
GOOGLE_PLACES_API_KEY=...
GOOGLE_CLIENT_ID=...
GOOGLE_CLIENT_SECRET=...
GOOGLE_REDIRECT_URI="${APP_URL}/auth/google/callback"

# Google Gemini (Assistant IA)
GEMINI_API_KEY=...
```

### 4. Migrations et build des assets
```bash
# Exécution des migrations du schéma
php artisan migrate

# Compilation des assets (Tailwind CSS v4 & Alpine)
npm run build
```

### 5. Démarrage
```bash
# Serveur web local Laravel
php artisan serve
```
