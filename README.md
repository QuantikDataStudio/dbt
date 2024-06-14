# Le jeu de données Airbnb
## Source: 
Le jeu de données a été téléchargé depuis le site https://insideairbnb.com/get-the-data/ qui regroupe les données Airbnb 
pour plusieurs villes. Pour notre travail, nous avons choisi la ville d'Amsterdam correspondant à un extrait du 11 Mars 2024 :). 

## Traitement
1. Division du fichier `listings.csv.gz` en 2 fichiers:
   1. [listings](dataset/listings.csv) avec un nombre de colonnes réduit et qui ne contient que les donneées qui 
   touchent directement au listing (i.e. on a enlevé les données de l'hôte et sur les revues) 
   2. [hosts](dataset/hosts.csv) ce fichier, extrait du fichier `listings.csv.gz`, ne contient que les infos concernant 
   l'hoôte. Ici aussi, nous avons limité le nombre de colonnes par rapport à toutes les infos qu'on avait
2. [reviews](dataset/reviews.csv) ce fichier a été téléchargé du jeu de données résumé où on n'a que 2 colonnes:
le `listing_id` et la `date` du commentaire qui a été laissé. Par exemple, ces 2 lignes ci-dessous
```csv
262394,2012-04-11
262394,2012-04-25
```
indiquent que le `listing_id` 262394 a reçu 2 commentaires: un le 11 Avril 2012 et l'autre le 25 Avril 2012.

# Mise en place de l'environnement

## Configuration de Snowflake
```sql
-- Utiliser le rôle admin
USE ROLE ACCOUNTADMIN;

-- Creer le rôle `transform` 
CREATE ROLE IF NOT EXISTS transform;
GRANT ROLE TRANSFORM TO ROLE ACCOUNTADMIN;

-- Créer la warehouse par défaut, si nécessaire 
CREATE WAREHOUSE IF NOT EXISTS COMPUTE_WH;
GRANT OPERATE ON WAREHOUSE COMPUTE_WH TO ROLE TRANSFORM;

-- Créer l'utilisateur DBT et lui assigner le rôle
CREATE USER IF NOT EXISTS dbt
  PASSWORD='MotDePasseDBT123@'
  LOGIN_NAME='dbt'
  MUST_CHANGE_PASSWORD=FALSE
  DEFAULT_WAREHOUSE='COMPUTE_WH'
  DEFAULT_ROLE='transform'
  DEFAULT_NAMESPACE='AIRBNB.RAW'
  COMMENT='Utilisateur DBT pour la transformation des données';
GRANT ROLE transform to USER dbt;

-- Création de la BDD et du schéma
CREATE DATABASE IF NOT EXISTS AIRBNB;
CREATE SCHEMA IF NOT EXISTS AIRBNB.RAW;

-- Mise en place des permissions pour le rôle `transform`
GRANT ALL ON WAREHOUSE COMPUTE_WH TO ROLE transform; 
GRANT ALL ON DATABASE AIRBNB to ROLE transform;
GRANT ALL ON ALL SCHEMAS IN DATABASE AIRBNB to ROLE transform;
GRANT ALL ON FUTURE SCHEMAS IN DATABASE AIRBNB to ROLE transform;
GRANT ALL ON ALL TABLES IN SCHEMA AIRBNB.RAW to ROLE transform;
GRANT ALL ON FUTURE TABLES IN SCHEMA AIRBNB.RAW to ROLE transform;
```

## Chargement des donneées
```sql
USE WAREHOUSE COMPUTE_WH;
    
create or replace api integration integration_jeu_de_donnees_github
api_provider = git_https_api
api_allowed_prefixes = ('https://github.com/QuantikDataStudio')
enabled = true;

create or replace git repository jeu_de_donnees_airbnb
api_integration = integration_jeu_de_donnees_github
origin = 'https://github.com/QuantikDataStudio/dbt.git';

create or replace file format format_jeu_de_donnees
type = csv
skip_header = 1
field_optionally_enclosed_by = '"';

CREATE TABLE AIRBNB.RAW.HOSTS
(
    host_id                STRING,
    host_name              STRING,
    host_since             DATE,
    host_location          STRING,
    host_response_time     STRING,
    host_response_rate     STRING,
    host_is_superhost      STRING,
    host_neighbourhood     STRING,
    host_identity_verified STRING
);


insert INTO AIRBNB.RAW.HOSTS (SELECT $1 as host_id,
                                     $2 as host_name,
                                     $3 as host_since,
                                     $4 as host_location,
                                     $5 as host_response_time,
                                     $6 as host_response_rate,
                                     $7 as host_is_superhost,
                                     $8 as host_neighbourhood,
                                     $9 as host_identity_verified
                              from @jeu_de_donnees_airbnb/branches/main/dataset/hosts.csv
                          (FILE_FORMAT => 'format_jeu_de_donnees'));

CREATE TABLE AIRBNB.RAW.LISTINGS
(
    id                     STRING,
    listing_url            STRING,
    name                   STRING,
    description            STRING,
    neighbourhood_overview STRING,
    host_id                STRING,
    latitude               STRING,
    longitude              STRING,
    property_type          STRING,
    room_type              STRING,
    accommodates           integer,
    bathrooms              FLOAT,
    bedrooms               FLOAT,
    beds                   FLOAT,
    amenities              STRING,
    price                  STRING,
    minimum_nights         INTEGER,
    maximum_nights         INTEGER
);

INSERT INTO AIRBNB.RAW.LISTINGS (SELECT $1  AS id,
                                        $2  AS listing_url,
                                        $3  AS name,
                                        $4  AS description,
                                        $5  AS neighbourhood_overview,
                                        $6  AS host_id,
                                        $7  AS latitude,
                                        $8  AS longitude,
                                        $9  AS property_type,
                                        $10 AS room_type,
                                        $11 AS accommodates,
                                        $12 AS bathrooms,
                                        $13 AS bedrooms,
                                        $14 AS beds,
                                        $15 AS amenities,
                                        $16 AS price,
                                        $17 AS minimum_nights,
                                        $18 AS maximum_nights
                                 from @jeu_de_donnees_airbnb/branches/main/dataset/listings.csv
                          (FILE_FORMAT => 'format_jeu_de_donnees'));


CREATE TABLE AIRBNB.RAW.REVIEWS
(
    listing_id  STRING,
    date        DATE
);

INSERT INTO AIRBNB.RAW.REVIEWS (SELECT $1 as listing_id,
                                       $2 as date
                                from @jeu_de_donnees_airbnb/branches/main/dataset/reviews.csv
                                    (FILE_FORMAT => 'format_jeu_de_donnees'));
```