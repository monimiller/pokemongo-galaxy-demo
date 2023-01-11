# pokemongo-galaxy-demo
This repository corresponds with the pokemon and mongo (get it?) demo done at
Trino Summit 11/10. If you want more in depth information on setup details,
please reach out to me. I'd love to send you more information to help you get
started and answer any questions. To federate my data sources, my MongoDB AWS region
is the same as my AWS S3 region, US East (N Virginia). Naming will be important
throughout the tutorial, so make sure you remember your names for you S3 bucket,
your catalogs, and your schemas.


## Pokemon(go) for the win

Hello fellow data friends. Welcome to this repository. The purpose of this
tutorial is to showcase the benefits of using Trino to create a data lakehouse
architecture within your data lake. We're using [Starburst
Galaxy](https://galaxy.starburst.io/login) because it's the easiest way to try
out Trino. What we will do is query data from MongoDB and S3, create a reporting
structure within our data lake, and then use the consume layer in our reporting
structure to create our Thoughtspot dashboard.

In this scenario, we’re going to query Pokemon pokedex data from MongoDB and
ingest Pokemon Go spawn location data which lands as raw CSV files in our data
lake. We then use Starburst Galaxy to read the data from the landing files,
clean up, and optimize that raw data into more performant ORC files in the
structure tables. The last step is to join and transform the spawn data and
pokedex data into a single table that is cleaned and ready to be utilized by a
data analyst, data scientist, or other data consumer. I then want to put on my
data analyst hat and do some visualizations to analyze the common types of
pokemon that spawn in San Francisco.


![Trino Summit Data Lakehouse Architecture
(1)](https://user-images.githubusercontent.com/33696269/200944562-20561b23-29e1-493b-92df-3fcd13fe3321.png)


## Add Pokedex to MongoDB

In this scenario, MongoDB holds the pokedex data. So it holds all the
information on the Pokemon such as it's unique number, it's Type 1, Type 2,
Abilities, Experience Type, Mega Evolution status, and much more. To query this
data, first insert this data into MongoDB. I'm using Atlas.

1. Download the [pokedex file](source-files/mongo-pokedex/pokedex.csv) from the repository.
2. Create a [MongoDB Atlas
   account](https://www.mongodb.com/cloud/atlas/register).
3. [Follow the
   instructions](https://www.mongodb.com/docs/atlas/getting-started/) to set up
   your Atlas account. I used Compass as a way to view my data.
4. Upload the pokedex CSV to MongoDB.
<img width="578" alt="Screen Shot 2022-11-08 at 9 59 22 PM"
src="https://user-images.githubusercontent.com/33696269/200935559-430fbda0-1d8d-4fe3-b577-99ec59dbd7de.png">

## Add Pokemon Go data to S3

We are imagining that our Pokemon Go data is getting streamed into our
data lake. For the purpose of this repo, we are using the CSV file as a snapshot
in our stream that we can manually upload to our data lake ourselves.

1. Create a S3 bucket with a descriptive name such as `pokemon-demo` . It would
   be best to add an initial identifier to the bucket such as `pokemon-demo-mm`.
   This will be used throughout the entire tutorial, so make sure you are aware 
   of your S3 bucket name and accurately replace it where appropriate. Use all the defaults.
2. Download the CSV containing the Pokemon Go data. {INSERT CSV}
3. Create three subfolders.
    - The first subfolder should hold the CSV containing the Pokemon Go data.
      Suggested name: `pokemon_spawns_csv`.
    - The second one should hold the Hive landing table that make up the land
      layer. Suggested name `pokemon-hive`.
    - The third one should hold the Iceberg tables that make up the structure
      and the consume layers. Suggested   name: `pokemon-iceberg`.
4. Upload the Pokemon Spawns CSV into the corresponding folder example path:`s3://pokemon-demo/pokemon-spawns-csv/pokemon-spawns.csv`
5. Create an AWS access key that will be used as the [authentication method for
   connecting from Starburst Galaxy to
   S3](https://docs.starburst.io/starburst-galaxy/security/external-aws.html)
   - Go to the IAM Management Console
   - Select *Users*
   - Select *Add Users*
   - Provide a Descriptive User Name like ```<username>-aws-pokemon```
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

For this portion, I will be letting Starburst Galaxy do the metastore
configurations for me. You also have the option to use Glue or Hive, which I
have also done in the past.

1. Navigate to the *Catalogs* tab. Click *Configure a Catalog*.

2. Create an S3 Catalog.
   - Catalog name: ```aws_pokemon``` or your pokemon of choice
   - Add a relevant description
   - Authenticate to S3 through the AWS Access Key/Secret Key created earlier
   - Metastore type: *Starburst Galaxy*
   - Default S3 bucket name: Enter the bucket you previously created. Example: `pokemon-demo`
   - Default directory name: Enter your selected default directory name.
     Example: `squirtle`
   - Enable *Allow creating external tables*
   - Enable *Allow writing to external tables*
   - Select default table format: *Iceberg*
   - Hit _Test connection_
   - Hit _Connect catalog_
   - Hit _Save access controls_ on the *Set Permissions* page
   - Select _Skip_ on the *Add to cluster* page

For more help configuring your AWS catalog, [visit the
documentation](https://docs.starburst.io/starburst-galaxy/catalogs/s3.html).

3.Create a MongoDB Catalog.
   - Catalog name: ```mongo_pokedex``` or your pokemon of choice
   - Add a relevant description
   - Authenticate to MongoDB using either a direct connection or via SSH tunnel.
     NOTE: If you have any special characters in your password, those may need
     to be coded properly. Using MongoDB Atlas and Compass, you can authenticate
     directly though your connection URL.

For more help configuring your MongoDB catalog, [visit the
documentation](https://docs.starburst.io/starburst-galaxy/catalogs/mongodb.html).

## Create a Starburst Galaxy Cluster

1. Navigate to the Clusters pane.
2. Click *Create a new cluster*
   - Enter cluster name: ```pokemon-cluster```
   - Catalogs: ```aws_pokemon``` & ```mongo_pokedex``` (select the catalogs
     previously created)
   - Cluster size: *Free*
   - Cluster type: *Standard*
   - Cloud provider region: *US East (N Virginia)* aka *us-east-1*
   - Hit _Create cluster_

That cluster will start up and once we add the necessary permissions and the
status changes to *Running* we will be ready to query! Navigate to the _Access control_ 
tab to get started.

## Add necessary admin permissions

1. Navigate to  the _Access control_ tab. Select _Roles and privileges_
2. Click into the highlighted *accountadmin* role
3. Select _Privileges_
4. Hit _Add privilege_
   - Modify privileges for: *Location*
   - Enter the S3 URI for your bucket. Add a _/*_ after the 
     location, this will allow you to access all the subfolders you created
     within that location. EX:_s3://pokemon-demo/*_

You are set up to get started! Navigate to the _Query editor_ tab.
## Navigate the query editor

Select the cluster you created in the top right corner. If named like above,
select ```pokemon-cluster```. Select your AWS catalog as the default catalog. If
named like above, select ```aws_pokemon``` .

## Create the land layer in the reporting structure

To create the land layer, you will first create a schema. Then we will create
the raw table from the CSV in S3.

1. Create schema in the pokemon hive bucket. ```CREATE SCHEMA landing_zone WITH
   (location = 's3://<AWS_BUCKET>/pokemon-hive/') ```
   example: ```CREATE SCHEMA landing_zone WITH
   (location = 's3://pokemon-demo/pokemon-hive/') ```
2. Create the landing table for the pokemon spawns. The external location should
   point to the location of your uploaded CSV file.

Update the CREATE TABLE statement to properly account for your own cluster,
catalog, and external location.
 ```sql
  CREATE TABLE aws_pokemon.landing_zone.pokemon_spawns(
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
  external_location = 's3://<S3 BUCKET NAME>/<CSV LOCATION>/',
  skip_header_line_count=1
); 
```
Example:
 ```sql
CREATE TABLE aws_pokemon.landing_zone.pokemon_spawns(
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
  external_location = 's3://pokemon-demo/pokemon-spawns-csv/',
  skip_header_line_count=1
);
```

Select from your table to validate that it's querying the right information.

```sql
SELECT * FROM aws_pokemon.landing_zone.pokemon_spawns LIMIT 10;
```
## Create the structure table in the reporting structure

We will create the structure table in all Iceberg format.  Since the pokedex
table is a dimension table containing pokemon attributes, I didn't feel the need
to copy it over into the land table. We will just build from here. 

1. Create the schema in the pokemon iceberg table. ```CREATE SCHEMA landing_zone
   WITH (location = 's3://pokemon-demo/pokemon_iceberg/') ```
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

Create the consume table. This will combine the Type 1, Type 2, and Mega
Evolution status from the pokedex for each pokemon found in the pokemon spawn
data. 

```sql
CREATE TABLE pokemon_spawns_by_type_2 AS
SELECT s.*, p.type1, p.type2, p.legendary, p.mega_evolution
FROM  pokemon_spawns p 
 JOIN pokemon s 
 ON p.number = s.number; 
 ```

## Build a dashboard on ThoughtSpot

I've downloaded the code to make this lightboard. If you want to play around
yourself, please reach out. I'll send it to you. 
