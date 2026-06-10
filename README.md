# actinvision-usecase-data -- Dataset — Use Case Power BI ActinVision

Repository public dédié à l'hébergement des fichiers de données utilisés par le dashboard Power BI réalisé dans le cadre du processus de recrutement [**ActinVision**](https://www.actinvision.com).

![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)
![Format](https://img.shields.io/badge/Format-Apache_Parquet-orange.svg)
![Compatible](https://img.shields.io/badge/Compatible-Power_BI-yellow.svg)

---

## Objectif

Ce repository sert de **source de données publique et stable** pour le fichier `Dashboard_Transactions_OUANKAP.pbix`. Le choix de GitHub comme hébergement répond à trois exigences :

- **Portabilité** — le `.pbix` peut être ouvert et actualisé depuis n'importe quelle machine
- **Reproductibilité** — sources accessibles sans authentification, URLs permanentes
- **Traçabilité** — versionnement Git, historique des modifications

## Architecture des données

```
Kaggle (3 CSV ~1.5 GB)
       │
       ▼
Python pandas + pyarrow
(nettoyage + conversion + compression)
       │
       ▼
GitHub public
(raw URLs + Release v1.0)
       │
       ▼
Power BI Desktop (.pbix)
```

## Source des données originales

Les données proviennent du dataset [**Financial Transactions Dataset: Analytics**](https://www.kaggle.com/datasets/computingvictor/transactions-fraud-datasets) publié sur Kaggle par `@computingvictor`, créé à l'origine par **Caixabank Tech** pour le AI Hackathon 2024.

Elles ont été converties du format CSV original au format **Apache Parquet** (compression Snappy) via un script Python de pré-traitement (`convertir_csv_parquet.py`), puis hébergées ici pour faciliter la consommation par Power BI.

## Contenu du repository

| Fichier | Hébergement | Description |
|---------|-------------|-------------|
| `users_data.parquet` | Repo (raw) | Référentiel clients : démographie, revenus, scoring crédit |
| `cards_data.parquet` | Repo (raw) | Référentiel cartes : marque, type, limite, dates clés |
| `transactions_data.parquet` | Release v1.0 | Historique des transactions filtré 2022-2024 |
| `convertir_csv_parquet.py` | Repo | Script de pré-traitement Python (CSV → Parquet) |
| `README.md` | Repo | Ce fichier |

## URLs de chargement Power BI

### Fichiers hébergés dans le repo (raw URLs)

```
https://github.com/Claude127/actinvision-usecase-data/raw/refs/heads/master/cards_data.parquet
https://github.com/Claude127/actinvision-usecase-data/raw/refs/heads/master/users_data.parquet
```

### Fichier hébergé en Release

```
https://github.com/Claude127/actinvision-usecase-data/releases/download/v1.0/transactions_data.parquet
```

### Formule Power Query (pour chaque fichier Parquet)

```m
let
    Source = Parquet.Document(
        Binary.Buffer(
            Web.Contents("https://raw.githubusercontent.com/TON_USER/TON_REPO/main/users_data.parquet")
        )
    )
in
    Source
```

> `Binary.Buffer` est obligatoire — sans cela, Power BI renvoie l'erreur "Parquet.Document cannot be used with streamed binary values".

## Pré-traitement appliqué

Les fichiers Parquet ont été produits à partir des CSV originaux via un script Python qui a appliqué :

- **Nettoyage des colonnes monétaires** : suppression du symbole `$` et des séparateurs de milliers, conversion en `float64`
- **Conversion des dates** : format `MM/YYYY` → date `YYYY-MM-01` (premier du mois)
- **Conversion des booléens** : harmonisation des valeurs `true/false`, `yes/no`, `1/0`
- **Catégorisation** : colonnes à faible cardinalité encodées en `category` pour optimiser la compression
- **Filtrage temporel** : transactions limitées à la période **2022–2024** pour maintenir un volume gérable

**Ratio de compression obtenu** : environ ×5 par rapport aux CSV originaux.

## Modélisation Power BI

### Schéma en étoile

```
              Dim_Client
                  │
                  │ (1 client → N transactions)
                  │
Dim_Calendrier ───┼─── Fait_Transactions ─── Dim_Commercant
                  │
                  │ (1 carte → N transactions)
                  │
              Dim_Carte
```

### Colonnes calculées clés ajoutées à Fait_Transactions

| Colonne | Calcul | Justification |
|---------|--------|---------------|
| `date_année_mois` | `FORMAT([date], "yyyy-MM")` | Permet le calcul du nombre de mois actifs par carte |
| `Z_Score` | `DIVIDE(amount - moyenne_client, ecart_type_client)` | Pré-calcule le score statistique pour éviter les explosions mémoire dans les mesures |
| `Est_Anormale` | `IF(ABS([Z_Score]) > 3, 1, 0)` | Drapeau binaire de transaction anormale |

## Choix méthodologiques importants

### 1. Périmètre du taux d'utilisation carte

L'analyse de **saturation** et d'**intensité d'utilisation** est restreinte aux cartes de type **Credit** avec une limite **≥ 1 000 USD**.

**Justifications :**
- Les cartes Debit et Debit (Prepaid) fonctionnent sur un solde provisionné, non sur une ligne de crédit revolving — le ratio dépense/limite n'a pas de signification métier pour ces types
- Les cartes avec credit_limit < 1 000 USD (~5% du panel) sont des outliers du dataset, vraisemblablement des données de test ou de saisie incorrectes

### 2. Nuance sur le "taux d'utilisation"

La mesure calculée est en réalité une **intensité mensuelle** (volume mensuel moyen rapporté à la limite). Elle diffère du "taux d'utilisation crédit" au sens réglementaire (encours moyen / limite), que les données disponibles ne permettent pas de calculer.

Un ratio > 100% n'indique pas nécessairement une saturation, mais reflète plusieurs cycles de paiement/remboursement par mois.

**Graduation en 4 niveaux :**
- **< 50%** : utilisation modérée
- **50–100%** : utilisation active
- **100–200%** : utilisation intense
- **≥ 200%** : utilisation très intense (à surveiller)

**Dualité avec la demande initiale :** la mesure `Carte Saturée` (seuil > 90%) est conservée pour respecter strictement le brief, et complétée par la classification `Niveau Intensité` comme valeur ajoutée analytique.

### 3. Détection d'activité anormale (Z-score)

Une transaction est marquée anormale si son montant s'écarte de plus de **3 écarts-types** de la moyenne du client (Z-score > 3 en valeur absolue). C'est un seuil statistique standard isolant ~0,3% des transactions.

**Choix d'implémentation :** le Z-score est stocké comme **colonne calculée** dans `Fait_Transactions` (calculé une fois au refresh) plutôt que comme mesure (recalculée à chaque visuel). Cela évite les erreurs de mémoire (`rsQueryMemoryLimitExceeded`) sur les volumes élevés de transactions.

## Structure du dashboard livré

| Page | Contenu | Mesures principales |
|------|---------|---------------------|
| 0. À propos | Contexte, source, architecture, contact | — |
| 1. Vue globale | KPIs volume, évolution temporelle, Top 5 commerçants | Total Transactions, Nb Transactions, Croissance MoM |
| 2. Vue client | Segmentation âge/revenu/région, ancienneté | Nb Clients, Montant Moyen par Client, Tranche d'âge |
| 3. Vue carte | KPIs cartes, répartition, cartes à risque, activité anormale | Carte Saturée, Niveau Intensité, Nb Transactions Anormales |
| 4. Dictionnaire | Définition de tous les indicateurs et leurs formules | — |

## Licence

Les données originales sont publiées par Caixabank Tech sous licence **Apache 2.0**. Cette republication respecte les termes de la licence d'origine.

## Auteur

**Claude Rowane DJIOJIP OUANKAP**

Étudiante ingénieure en Systèmes d'Information — Préparation du MS Expert Big Data Engineer à l'Université de Technologie de Troyes (UTT)

- Email : ouankaprowane@gmail.com
- LinkedIn : [linkedin.com/in/rowaneouankap](https://linkedin.com/in/rowaneouankap)

---

*Repository créé dans le cadre du processus de recrutement ActinVision — juin 2026. Pour toute question, ouvrir une [issue](../../issues).*
