# pokemongo-galaxy-demo
This repository corresponds with the pokemon and mongo (get it?) demo done at Trino Summit 11/10. If you want more in depth information on setup details, please reach out to me. I'd love to send you more information and help you get started. 


## Pokemon(go) for the win

Hello fellow data friends. Welcome to this repository. The purpose of this tutorial is to showcase the benefits of using Trino to create a data lakehouse architecture within your data lake. We're using [Starburst Galaxy](https://galaxy.starburst.io/login) because it's the easiest way to try out Trino. What we will do is query data from MongoDB and S3, create a reporting structure within our data lake, and then use the consume layer in our reporting structure to create our Thoughtspot dashboard. 

In this scenario, we’re going to query Pokemon pokedex data from MongoDB and ingest Pokemon Go spawn location data which lands as raw CSV files in our data lake. We then use Starburst Galaxy to read the data from the landing files, clean up, and optimize that raw data into more performant ORC files in the structure tables. The last step is to join and transform the spawn data and pokedex data into a single table that is cleaned and ready to be utilized by a data analyst, data scientist, or other data consumer. I then want to put on my data analyst hat and do some visualizations to analyze the common types of pokemon that spawn in San Francisco. 


![Trino Summit Data Lakehouse Architecture (1)](https://user-images.githubusercontent.com/33696269/200944562-20561b23-29e1-493b-92df-3fcd13fe3321.png)


## Add Pokedex to MongoDB

In this scenario, MongoDB holds the pokedex data. So it holds all the information on the Bulbasaur pokemon such as it's unique number, it's Type 1, Type 2, Abilities, Experience Type, Mega Evolution status, and much more.  Insert this data into mongodb. I'm using Atlas. 

1. Download the pokedex file from the repository. [INSERT FILE]
2. Create a [MongoDB Atlas account](https://www.mongodb.com/cloud/atlas/register).
3. [Follow the instructions](https://www.mongodb.com/docs/atlas/getting-started/) to set up your Atlas account. I used Compass as a way to view my data.  
4. Upload the pokedex CSV to MongoDB. 
5. 
<img width="578" alt="Screen Shot 2022-11-08 at 9 59 22 PM" src="https://user-images.githubusercontent.com/33696269/200935559-430fbda0-1d8d-4fe3-b577-99ec59dbd7de.png">


## Add Pokemon Go data to S3

We imagine that our Pokemon Go data is potentially getting streamed into our data lake. For the purpose of this repo, we are using the CSV file as a snapshot in our stream that we can manually upload to our data lake ourselves. 

1. Create a S3 bucket with a descriptive name such as ```pokemon-demo``` . Use all the defaults.
2. Download the CSV containing the Pokemon Go data. {INSERT CSV}
3. Create three subfolders. 
    - The first subfolder should hold the CSV contaning the Pokemon Go data. Suggested name: `pokemon_spawns`.
    - The second one should hold the Hive landing table that make up the land layer. Suggested name `pokemon_hive`.
    - The third one should hold the Iceberg tables that make up the structure and the consume layers. Suggested   name: `pokemon_iceberg`.
4. Upload the Pokemon Spawns CSV into the corresponding folder.
5. Create an AWS access key that will be used as the
   [authentication method for connecting from {{site.terms.sg}} to
   S3](https://docs.starburst.io/starburst-galaxy/security/external-aws.html)
   - Go to the IAM Management Console
   - Select *Users*
   - Select *Add Users*
   - Provide a Descriptive User Name like ```<username>-aws-covid```
   - Select AWS Credential Type: *Access key - Programmatic access*
   - Set Permissions: *Attach existing policies directly*
   - Add the following policy: *AmazonS3FullAccess*

6. Finish creating the access key with the rest of the defaults, and then save
   your AWS Access Key and Secret Access Key.
   
## Create a Starburst Galaxy account

1. Navigate to the [Starburst Galaxy login
   page](https://galaxy.starburst.io/login).

2. If you already don't have an account, create a new one and verify the account
   with your email.

## Create Starburst Galaxy Catalogs

1. Navigate to the *Catalogs* tab. Click *Configure a Catalog*.

2. Create an S3 Catalog.
   - Catalog name: ```aws_pokemon``` or your pokemon of choice
   - Add a relevant description
   - Authenticate to S3 through the AWS Access Key/Secret created earlier
   - Metastore configuration: *"I don't have a metastore"*
   - Default directory name: ```<username>_metadata```
   - Enable *Allow creating external tables*
   - Enable *Allow writing to external tables*
   - Select default table format: *Iceberg*
   - Hit _Skip_ on the *Set Permissions* page

For more help configuring your AWS catalog, [LINK]visit the documentation.

3.Create a MongoDB Catalog.
   - Catalog name: ```mongo_pokedex``` or your pokemon of choice
   - Add a relevant description
   - Authenticate to MongoDB using either a direct connection or via SSH tunnel. NOTE: If you have any special characters in your password, those may need to be coded properly. 

For more help configuring your MongoDB catalog, {LINK} visit the documentation.

## Create a Starburst Galaxy Cluster

1. Navigate to the Clusters pane.  
2. Click *Create a new cluster*
   - Enter cluster name: ```<username>-pokemon```
   - Cluster size: *Free*
   - Cluster type: *Standard*
   - Catalogs: ```aws_pokemon``` & ```mongo_pokedex``` (select the catalogs previously created)
   - Cloud provider region: *US East (N Virginia)* aka *us-east-1*

## Create the land layer in the reporting structure

To create the land layer, you will first create a schema. Then we will create the raw table from the CSV in S3. 

1. Create schema in the pokemon hive bucket. ```CREATE SCHEMA landing_zone WITH (location = 's3://pokemon-demo/pokemon_hive/') ```
2. Create the landing table for the pokemon spawns. The external location should point to the location of your uploaded CSV file.
 ```sql
  CREATE TABLE pokemon_spawns_land(
  "s2_id" VARCHAR,
  "s2_token" VARCHAR,
  "num" VARCHAR,
  "name" VARCHAR,
  "lat" VARCHAR,
  "long" VARCHAR,
  "encounter_ms" VARCHAR,
  "disappear_ms" VARCHAR
)
WITH (
  type = 'HIVE',
  format = 'CSV',
  external_location = 's3://pokemon-demo/pokemon_spawns/csv/',
  skip_header_line_count=1
); 
```

## Create the structure table in the reporting structure

We will create the structure table in all Iceberg format.  Since the pokedex table is a dimension table containing pokemon attributes, I didn't feel the need to copy it over into the land table. We will just build from here. 

1. Create the schema in the pokemon iceberg table. ```CREATE SCHEMA landing_zone WITH (location = 's3://pokemon-demo/pokemon_iceberg/') ```
2. Create the structure table for the pokemon go spawn data stored in S3. 

```sql
CREATE TABLE pokemon
WITH (
  format = 'ORC',
  location = 's3://pokemon-mm/structure/pokemon/orc'
) AS
SELECT
  CAST(num AS INTEGER) AS number,
  name,
  CAST(lat AS DOUBLE) AS lat,
  CAST(long AS DOUBLE) AS long,
  CAST(encounter_ms AS BIGINT) AS encounter_ms,
  CAST(disappear_ms AS BIGINT) AS disappear_ms
FROM aws_ketchum.catch_em_all.pokemon_csv;
```

4. Create the structure table for the pokemon pokedex data stored in MongoDB.

```sql
CREATE TABLE pokemon_pokedex_structure 
WITH (
  format = 'ORC'
) AS
SELECT
  CAST(number AS INTEGER) AS number,
  name,
  "Type 1" AS type1,
  "Type 2" AS type2,
  CAST(json_parse(replace(replace(Abilities, '''s', 's'), '''', '"')) AS ARRAY(VARCHAR)) AS abilities,
  CAST(hp AS INTEGER) AS hp,
  CAST(att AS INTEGER) AS att,
  CAST(def AS INTEGER) AS def,
  CAST(spa AS INTEGER) AS spa,
  CAST(spd AS INTEGER) AS spd,
  CAST(spe AS INTEGER) AS spe,
  CAST(bst AS INTEGER) AS bst,
  CAST(mean AS DOUBLE) AS mean,
  CAST("Standard Deviation" AS DOUBLE) AS std_dev,
  CAST(generation AS DOUBLE) AS generation,
  "Experience type" AS experience_type,
  CAST("Experience to level 100" AS BIGINT) AS experience_to_lvl_100,
  CAST(CAST("Final Evolution" AS DOUBLE) AS BOOLEAN) AS final_evolution,
  CAST("Catch Rate" AS INTEGER) AS catch_rate,
  CAST(CAST("Legendary" AS DOUBLE) AS BOOLEAN) AS legendary,
  CAST(CAST("Mega Evolution" AS DOUBLE) AS BOOLEAN) AS mega_evolution,
  CAST(CAST("Alolan Form" AS DOUBLE) AS BOOLEAN) AS alolan_form,
  CAST(CAST("Galarian Form" AS DOUBLE) AS BOOLEAN) AS galarian_form,
  CAST("Against Normal" AS DOUBLE) AS against_normal,
  CAST("Against Fire" AS DOUBLE) AS against_fire,
  CAST("Against Water" AS DOUBLE) AS against_water,
  CAST("Against Electric" AS DOUBLE) AS against_electric,
  CAST("Against Grass" AS DOUBLE) AS against_grass,
  CAST("Against Ice" AS DOUBLE) AS against_ice,
  CAST("Against Fighting" AS DOUBLE) AS against_fighting,
  CAST("Against Poison" AS DOUBLE) AS against_poison,
  CAST("Against Ground" AS DOUBLE) AS against_ground,
  CAST("Against Flying" AS DOUBLE) AS against_flying,
  CAST("Against Psychic" AS DOUBLE) AS against_psychic,
  CAST("Against Bug" AS DOUBLE) AS against_bug,
  CAST("Against Rock" AS DOUBLE) AS against_rock,
  CAST("Against Ghost" AS DOUBLE) AS against_ghost,
  CAST("Against Dragon" AS DOUBLE) AS against_dragon,
  CAST("Against Dark" AS DOUBLE) AS against_dark,
  CAST("Against Steel" AS DOUBLE) AS against_steel,
  CAST("Against Fairy" AS DOUBLE) AS against_fairy,
  CAST("Height" AS DOUBLE) AS height,
  CAST("Weight" AS DOUBLE) AS weight,
  CAST("BMI" AS DOUBLE) AS bmi
FROM mongodb.pokemongo.pokedex;
```

## Create the consume table in the reporting structure

Create the consume table. This will combine the Type 1, Type 2, and Mega Evolution status from the pokedex for each pokemon found in the pokemon spawn data. 

```sql
CREATE TABLE pokemon_spawns_by_type_2 AS
SELECT s.*, p.type1, p.type2, p.legendary, p.mega_evolution
FROM  pokemon_spawns p 
 JOIN pokemon s 
 ON p.number = s.number; 
 ```

## Build a dashboard on ThoughtSpot

I've downloaded the code to make this lightboard. If you want to play around yourself, please reach out. I'll send it to you. 
