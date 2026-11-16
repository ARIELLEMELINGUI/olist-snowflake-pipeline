# Olist : pipeline de données avec Snowflake (bronze / silver / gold)

Mini-projet de data engineering de bout en bout, construit pour s'entraîner sur Snowflake à partir d'un jeu de données e-commerce réel.

## Question métier

1. Quelles **catégories** et quelles **régions** rapportent le plus ?
2. Les **retards de livraison** sont-ils associés à une baisse de la satisfaction client ?

## Données

Jeu de données public **Olist** (Kaggle) : une place de marché brésilienne, environ 100 000 commandes entre 2016 et 2018, réparties en 9 fichiers CSV. 8 fichiers utilisés (le fichier de géolocalisation, très lourd et inutile ici, est écarté). Licence non commerciale.

## Architecture

```
CSV Olist  →  BRONZE (brut)  →  SILVER (nettoyé)  →  GOLD (étoile)  →  requêtes d'analyse
              Snowflake          Snowflake            Snowflake
```

| Couche | Rôle | Choix principaux |
|---|---|---|
| **BRONZE** | Données telles quelles | Toutes les colonnes en texte, plus le fichier d'origine et la date de chargement (traçabilité). Chargement par `COPY INTO` depuis un stage. |
| **SILVER** | Données fiables | Typage avec `TRY_TO_TIMESTAMP` / `TRY_TO_DECIMAL` (une valeur illisible devient `NULL` au lieu de faire échouer la requête), nettoyage (`TRIM`, `LOWER`/`UPPER`), dédoublonnage avec `QUALIFY ROW_NUMBER()`, catégories traduites en anglais. |
| **GOLD** | Prêt pour le métier | Schéma en étoile : `FACT_ORDER_ITEMS` (une ligne par article, avec retard et note) et dimensions client, produit, vendeur, date. |

Infrastructure : un warehouse `XSMALL` qui s'éteint après 60 secondes d'inactivité, une base `OLIST_DB` et trois schémas. Tous les objets sont référencés par leur nom complet (`base.schéma.objet`).

## Contrôles qualité

- Nombre de lignes identique entre BRONZE et SILVER pour les 8 tables.
- Nombre de lignes de la table de faits identique à `SILVER.ORDER_ITEMS` (112 650), pour détecter toute duplication par jointure.
- Intégrité entre tables : aucun article sans commande, produit ou vendeur, aucune commande sans client.
- Aucun prix nul ou négatif, aucune livraison antérieure à l'achat.
- Piège identifié : `customer_id` change à chaque commande, le vrai client est `customer_unique_id`.
- Avis : `review_id` n'est pas unique, donc les avis sont agrégés par commande avant la jointure.

## Résultats

**Retard et satisfaction** (commandes livrées) :

| | Commandes | Note moyenne |
|---|---|---|
| À l'heure | 88 644 | 4,21 |
| En retard | 7 826 | 2,55 |

Environ 8 % des commandes livrées arrivent en retard, avec une note inférieure d'environ 1,7 point.

**Régions** : le sud-est domine. São Paulo (SP) génère à elle seule environ 5,2 M de chiffre d'affaires, devant Rio de Janeiro (RJ, environ 1,8 M) et Minas Gerais (MG, environ 1,6 M).

**Catégories** : en tête, `health_beauty` (environ 1,26 M), `watches_gifts` (environ 1,20 M) et `bed_bath_table` (environ 1,04 M).

Définitions : le chiffre d'affaires est la somme des prix des articles (hors frais de port), sur les commandes non annulées. La région est celle du client.

## Limites

- Les retards sont **associés** à une note plus basse : c'est une corrélation, pas une preuve de causalité.
- 1 076 commandes (environ 1 %) présentent un écart entre le total payé et le total des articles (bons de réduction, commandes sans article, arrondis). Documenté, non corrigé.
- 8 commandes marquées livrées n'ont pas de date de livraison : elles sont exclues du calcul du retard.
- Données de 2016 à 2018 et licence non commerciale.

## Prochaines étapes

- Mise en miroir des tables GOLD dans Microsoft Fabric (OneLake), puis rapport Power BI.
- Orchestration du chargement (tâches Snowflake) pour automatiser le pipeline.
