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
Pour configurer l'accès de DBT à Snowflake, copiez-coller cet ensemble de requêtes SQL dans Snowflake et exécutez-le. 
N'hésitez pas à changer le mot de passe pour plus de sécurité.  
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
Pour charger les données dans Snowflake, il nous faut faire un peu de gymnastique SQL. Nous vous proposons de copier-coller
cet ensemble de requêtes et de les exécuter dans Snowflake. Vous devriez trouver l'explication de chaque ligne dans le cours.

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


# Changement du schéma curation

```jinja
{% macro generate_schema_name(custom_schema_name, node) -%}

    {%- set default_schema = target.schema -%}
    {%- if custom_schema_name is none -%}

        {{ default_schema }}

    {%- else -%}

        {{ custom_schema_name | trim }}

    {%- endif -%}

{%- endmacro %}
```

# Création des 1ers modèles

Le SQL pour `curation_hosts` est le suivant:

```snowflake
WITH hosts_raw AS (
    SELECT
		host_id,
		CASE WHEN len(host_name) = 1 THEN 'Anonyme' ELSE host_name END AS host_name,
		host_since,
		host_location,
		SPLIT_PART(host_location, ',', 1) AS host_city,
		SPLIT_PART(host_location, ',', 2) AS host_country,
		TRY_CAST(REPLACE(host_response_rate, '%', '') AS INTEGER) AS response_rate,
		host_is_superhost = 't' AS is_superhost,
		host_neighbourhood,
		host_identity_verified = 't' AS is_identity_verified
    FROM airbnb.raw.hosts)
SELECT *
from hosts_raw;
```

Le SQL pour `curation_listings` est le suivant:
```snowflake
WITH listings_raw AS 
	(SELECT 
		id AS listing_id,
		listing_url,
		name,
		description,
		description IS NOT NULL has_description,
		neighbourhood_overview,
		neighbourhood_overview IS NOT NULL AS has_neighrbourhood_description,
		host_id,
		latitude,
		longitude,
		property_type,
		room_type,
		accommodates,
		bathrooms,
		bedrooms,
		beds,
		amenities,
        try_cast(split_part(price, '$', 1) as float) as price
		minimum_nights,
		maximum_nights
	FROM airbnb.raw.listings )
SELECT *
FROM listings_raw
```

# Définir les sources

```yaml
version: 2

sources:
  - name: raw_airbnb_data
    database: airbnb  
    schema: raw  
    tables:
      - name: hosts
      - name: listings
      - name: reviews

```

# Définir la seed
Les données sur le nombre de visiteurs à Amsterdam par an sont tireées du site https://opendata.cbs.nl/#/CBS/en/dataset/82061ENG/table?searchKeywords=amsterdam 

Ajouter ce code à votre fichier `dbt_project.yaml`: 
```yaml
seeds:
  analyse_airbnb:
    tourists_per_year:
      +enabled: true
      +database: airbnb
      +schema: raw
```

Pour créer la table `airbnb.curation.tourists_per_year`, on utilise le SQL suivant:

```sql
with tourists_per_year as (
    SELECT year, tourists
   from ref("tourists_per_year")
)
SELECT
    DATE(year || '-12-31') as year,
    tourists 
from tourists_per_year    
```

# Définir les snapshot

Définir le snapshot pour la table `hosts`
```jinja
{% snapshot hosts_snapshot %}

    {{
        config(
          target_database='airbnb',
          target_schema='snapshots',
          strategy='check',
          check_cols='all',
          unique_key='host_id'
        )
    }}

    select * from {{ source('raw_airbnb_data', 'hosts') }}

{% endsnapshot %}
```

Pour modifier une ligne dans la table `airbnb.raw.hosts`, on peut utiliser cette requête SQL: 
```sql
UPDATE airbnb.raw.hosts SELECT host_response_time='within an hour', host_response_rate='100%'
where host_id='1376607';
```

# Tests de qualité de données et unitaires
## Tests de qualité de donnée

### Test de la qualité des données dans sources 
```yaml
version: 2

sources:
  - name: raw_airbnb_data
    database: airbnb
    schema: raw
    tables:
      - name: hosts
        columns:
          - name: host_id
            tests:
              - unique
              - not_null

      - name: listings
      - name: reviews
```

### Test de la qualité des données des modèles
```yaml
version: 2

models:
  - name: curation_hosts
    description: Table hotes nettoyée et formatée
    columns:
      - name: host_id
        description: Identifiant unique de l'hôte
        tests:
          - unique
          - not_null
      - name: host_name
        description: Nom de l'hôte
        tests:
          - not_null
      - name: host_since
        description: Date d'inscription de l'hôte
        tests:
          - not_null
      - name: host_location
        description: Ville et pays de l'hôte
        tests:
          - not_null
      - name: host_city
        description: Ville de l'hôte
        tests:
          - not_null
      - name: host_country
        description: Pays de l'hôte
        tests:
          - not_null
      - name: is_superhost
        description: Indicateur si l'hôte a le statut superhost 
        tests:
          - not_null
          - accepted_values:
              values: [TRUE, FALSE]
      - name: host_neighbourhood
        description: Quartier de l'hôte
        tests:
          - not_null
      - name: is_identity_verified
        description: Indicateur si l'identité de l'hôte a été vérifiée  
        tests:
          - not_null
          - accepted_values:
              values: [TRUE, FALSE]
```

## Tests unitaires pour SQL
À ajouter au fichier `schema.yaml`
```yaml
unit_tests:
  - name: test_is_host_data_transformation_correct
    description: "Vérifie que host_name, host_city, host_country et response_rate sont créés correctement"
    model: curation_hosts
    given:
      - input: ref('hosts_snapshot')
        rows:
          - {host_name: 'Jacko', host_location: "ville,pays", host_response_rate: '32%'}
          - {host_name: 'Xi', host_location: "ville,pays", host_response_rate: '32%'}
          - {host_name: 'J', host_location: "pays,ville", host_response_rate: '32.53%'}
    expect:
      rows:
        - {host_name: 'Jacko', host_city: 'ville', host_country: 'pays', response_rate:32}
        - {host_name: 'Xi', host_city: 'ville', host_country: 'pays', response_rate:32}
        - {host_name: 'Anonyme', host_city: 'pays', host_country: 'ville', response_rate: 32}

```

```yaml
unit_tests:
  - name: test_is_host_data_transformation_correct
    description: "Vérifie que host_name, host_city, host_country et response_rate sont créés correctement"
    model: curation_hosts
    given:
      - input: ref('hosts_snapshot')
        rows:
          - {host_name: 'Jacko', host_location: "ville,pays", host_response_rate: '32%', DBT_VALID_TO: null, host_is_superhost: 't', host_neighbourhood: 'quartier'} 
          - {host_name: 'Xi', host_location: "ville,pays", host_response_rate: '32.03%', DBT_VALID_TO: null, host_is_superhost: 't', host_neighbourhood: 'quartier'}
          - {host_name: 'J', host_location: "pays,ville", host_response_rate: '32.53%', DBT_VALID_TO: null, host_is_superhost: 't', host_neighbourhood: 'quartier'}
    expect:
      rows:
        - {host_name: 'Jacko', host_city: 'ville', host_country: 'pays', response_rate: 32}
        - {host_name: 'Xi', host_city: 'ville', host_country: 'pays', response_rate: 32}
        - {host_name: 'Anonyme', host_city: 'pays', host_country: 'ville', response_rate: 33}
```

## Tests unitaires pour macros
```yaml
{% macro extraire_prix_a_partir_dun_caractere(price, symbol) -%}
    try_cast(
        CASE 
            WHEN STARTSWITH({{ price }}, '{{ symbol }}') THEN SPLIT_PART({{ price }}, '{{ symbol }}', 2)
            WHEN ENDSWITH({{ price }}, '{{ symbol }}') THEN SPLIT_PART({{ price }}, '{{ symbol }}', 1)
            ELSE NULL
        END
    AS FLOAT)
{% endmacro %}
```

## Installation de DBT utils
Créer le fichier `packages.yaml` avec le contenu suivant

```yaml
packages:
  - package: dbt-labs/dbt_utils
    version: 1.2.0
 ```

```yaml
- name: minimum_nights
  tests:
    - dbt_utils.accepted_range:
        min_value: 1
- name: price
  tests:
     - dbt_utils.accepted_range:
        min_value: 0
        inclusive: false
```

# Materialisation incrementale

## Modification de la table hosts
```sql
ALTER TABLE AIRBNB.RAW.HOSTS ADD COLUMN LOAD_TIMESTAMP TIMESTAMP;
UPDATE AIRBNB.RAW.HOSTS SET LOAD_TIMESTAMP = CURRENT_TIMESTAMP;
SELECT * from AIRBNB.raw.hosts limit 10;
```

## Creation du model hosts_inc
```jinja
{{
    config(
        database=var('inc_database'),
        schema=var('inc_schema'),
        materialized='incremental',
        unique_key='host_id'
    )
}}
with raw_hosts as (
    SELECT * from {{ source("raw_airbnb_data", "hosts") }}
)
select *
from raw_hosts
{% if is_incremental() %}
where load_timestamp > (select max(load_timestamp) from {{this}} )
{% endif %}
```

N'oubliez pas de définir les valeurs des variables au niveau du projet, dans le fichier dbt_project.yml :)

## Insérer une nouvelle ligne dans raw.hosts
```sql
SELECT * from airbnb.raw.hosts where host_id='1376607';

insert into airbnb.raw.hosts VALUES (
'1376607','Martin S','2011-11-06'::DATE,'Amsterdam, Netherlands','within an hour','100%','t','Hoofddorppleinbuurt','t',current_timestamp
)
```

## rajouter ce SQL pour éviter les duplicate 
```sql
{% else %}
join (select host_id, max(load_timestamp) as load_timestamp from {{ source("raw_airbnb_data", "hosts") }} group by 1) t2
on raw_hosts.host_id = t2.host_id and raw_hosts.load_timestamp = t2.load_timestamp
```


# Projet final
- Distribution des prix par quartier
- Distribution des super hotes par quartier
- Relation prix <> super hote

- hypothese:
-   - 1 airbnb => 1 review 
-   - 1 airbnb => 0.8 review
- est-ce qu'il y a plus/moins de visiteurs à Amsterdam qui préferent des nuits dans un airbnb par rapport à l'hôtel
