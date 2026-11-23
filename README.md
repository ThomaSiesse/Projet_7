# Projet 7 : concevoir et analyser une base de données NoSQL

Restauration et analyse d'une base **MongoDB** d'annonces de locations de courte durée (Airbnb) à Paris et à Lyon, puis mise en place d'une architecture distribuée (réplication et sharding) pour pérenniser la base.

## Contexte

L'association **Nos Cités** étudie l'impact des plateformes de location de courte durée sur le logement à Lyon et à Paris. Les données étaient stockées en local sur des serveurs à Paris et à Lyon, et **la base de Paris a crashé**.

La mission se déroule en trois parties :

1. **Restaurer** la base de données à partir des fichiers d'annonces ;
2. **Analyser** les données avec des requêtes pour vérifier leur intégrité et répondre aux questions de l'association ;
3. **Concevoir** une architecture plus robuste, avec réplication et sharding, pour pérenniser la base.

## Contenu du dossier

| Élément | Description |
| --- | --- |
| `clean_data.ipynb` | Ajoute la colonne `ville`, supprime les colonnes de texte long et d'URL, nettoie le texte et convertit `price` en nombre |
| `insert_data.ipynb` | Fusionne Paris et Lyon et insère les 105 858 annonces dans MongoDB (base et collection `logement`) |
| `Untitled.ipynb` | Requêtes d'analyse avec pymongo et Polars (taux de réservation, avis, densité par quartier…) |
| `Untitled1.ipynb` | Premier nettoyage du fichier de Paris (export `logements_clean.csv`) |
| `listings_Paris.csv`, `listings_Lyon.csv` | Données brutes des annonces (format Inside Airbnb) |
| `paris_with_ville.csv`, `lyon_with_ville.csv` | Données brutes avec la colonne `ville` |
| `paris_clean.csv`, `lyon_clean.csv`, `logements_clean.csv` | Données nettoyées, prêtes à être importées |
| `replicaset/` | Répertoires de données des instances MongoDB du cluster |
| `restaurants/`, `tourPedia_paris.json` | Jeux de données d'exercice du cours MongoDB, hors livrables |
| `Concevez_et_analysez_une_base_de_données_NoSQL_SIESSE_thomas/` | Support de soutenance (`.pptx`) |
| `Concevez_et_analysez_une_base_de_données_NoSQL_SIESSE_thomas.zip` | Archive des livrables, prête à être déposée |

## Prérequis

- MongoDB (`mongod`, `mongos`, `mongosh`)
- Python 3 avec `pymongo`, `pandas` et `polars`
- Jupyter

```bash
pip install pymongo pandas polars jupyterlab
```

## Partie 1 : restauration de la base

1. **Nettoyage** (`clean_data.ipynb`) :
   - ajout d'une colonne `ville` (`Paris` ou `Lyon`) ;
   - suppression des colonnes inutiles pour l'analyse (`description`, `neighborhood_overview`, `host_about` et les URL d'images et de pages) ;
   - suppression des guillemets, retours à la ligne et virgules dans les champs texte ;
   - conversion de `price` (`"$1,250.00"`) en nombre décimal.
2. **Insertion** (`insert_data.ipynb`) : concaténation des deux villes, puis insertion avec `insert_many` dans la collection `logement`. Au total, **105 858 documents** sont insérés.

Le NoSQL a été retenu pour sa souplesse de schéma, sa capacité à répartir les données sur plusieurs serveurs et la rapidité des requêtes.

## Partie 2 : analyse

Analyses réalisées sur la collection `logement` :

- nombre d'annonces par type de location ;
- les 5 annonces les mieux évaluées ;
- nombre d'hôtes différents ;
- nombre et proportion de logements réservables instantanément ;
- hôtes possédant plus de 100 biens, et leur part dans le total ;
- nombre et proportion de super-hôtes ;
- taux de réservation par mois et par type de logement ;
- médiane du nombre d'avis, globale et par catégorie d'hôte ;
- densité de logements par quartier de Paris ;
- quartiers avec le plus fort taux de réservation.

Le taux de réservation est estimé à partir de la disponibilité sur 30 jours :

```python
taux_reservation = (30 - availability_30) / 30
```

Exemple : taux de réservation moyen par quartier avec Polars.

```python
(df.with_columns(((30 - pl.col("availability_30")) / 30).alias("taux_reservation"))
   .group_by("neighbourhood_cleansed")
   .agg(pl.col("taux_reservation").mean().alias("taux_moyen"),
        pl.len().alias("nombre_logements"))
   .sort("taux_moyen", descending=True)
   .head(10))
```

## Partie 3 : architecture distribuée

Le dossier `replicaset/` contient les répertoires de données des instances MongoDB du cluster :

| Répertoire | Rôle |
| --- | --- |
| `c1`, `c2`, `c3` | Membres d'un replica set |
| `arb` | Arbitre du replica set (vote lors de l'élection d'un primaire) |
| `config1`, `config2` | Serveurs de configuration du cluster shardé |
| `shard1`, `shard2` | Shards (partitions des données) |
| `R0S1` | Instance de test |

La réplication assure la disponibilité : si le primaire tombe, un secondaire prend le relais. Le sharding répartit les documents sur plusieurs serveurs pour absorber la volumétrie.

Les données sont insérées via le point d'entrée du cluster, sur le port `27040` :

```python
client = MongoClient("mongodb://localhost:27040/")
collection = client["logement"]["logement"]
```

## Remarques

- Les fichiers CSV (jusqu'à 190 Mo) et `replicaset/` sont volumineux. Ils ne doivent pas être versionnés avec Git.
- Les données d'annonces proviennent d'Inside Airbnb.

## Auteur

Thomas SIESSE, juin 2026.
