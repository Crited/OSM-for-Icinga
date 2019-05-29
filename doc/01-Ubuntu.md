# Manually building a tile server on Ubuntu

This page describes how to setup everything that is required to operate your own *tile server*. These instructions were tested on *Ubuntu Linux 18.04 LTS (Bionic Beaver)*. 

Pretty much everything that's written here is based on: **[switch2osm](https://switch2osm.org/manually-building-a-tile-server-18-04-lts/)**

#### Table of Contents

1. [Dependencies](#Dependencies)
2. [postgreSQL](#postgreSQL)
3. [osm2pgSQL](#osm2pgSQL)
4. [Mapnik](#Mapnik)
5. [Renderd](#Renderd)
6. [Stylesheet](#Stylesheet)
7. [Data](#Data)
8. [Shapefile](#Shapefile)
9. [Fonts](#Fonts)
10. [Configuration](#Configuration)
11. [Apache](#Apache)
12. [Run](#Run)
13. [Service](#Service)
14. [Adding](#adding)
15. [Merging](#merging)

## Dependencies

This *[OSM](https://www.openstreetmap.org/#map=3/19.97/10.11) tile server* setup is build around five major parts: *[mod_tile](https://wiki.openstreetmap.org/wiki/Mod_tile); renderd; [mapnik](https://wiki.openstreetmap.org/wiki/Mapnik); [osm2pgsql](https://wiki.openstreetmap.org/wiki/Osm2pgsql); [postgis](https://wiki.openstreetmap.org/wiki/PostGIS)*.
I'll explain these as we get to it. In order to install all of these components, some dependencies *have* to be resolved first.
```sh
sudo apt install libboost-all-dev git-core tar unzip wget bzip2 build-essential autoconf libtool libxml2-dev libgeos-dev libgeos++-dev libpq-dev libbz2-dev libproj-dev munin-node munin libprotobuf-c0-dev protobuf-c-compiler libfreetype6-dev libtiff5-dev libicu-dev libgdal-dev libcairo-dev libcairomm-1.0-dev apache2 apache2-dev libagg-dev liblua5.2-dev ttf-unifont lua5.1 liblua5.1-dev libgeotiff-epsg curl
```
Later on we will need the osmadmins-repository might aswell add it now:
```sh
sudo add-apt-repository ppa:osmadmins/ppa
sudo apt-get update
```

## postgreSQL

*PostgreSQL* (or just *"Postgres"*) is an object-relational database, which we'll be using to store our *OSM data*. *Postgis* is an extension for *Postgres*. PostGIS adds additional geospatial types and functions which make handling spatial data in the database easier and more powerful. Basically it's perfect for us. Both are available pre-packaged on Ubuntu.
```sh
sudo apt-get install postgresql postgresql-contrib postgis postgresql-10-postgis-2.4 postgresql-10-postgis-scripts
```
Now that *Postgres* and *Postgis* are installed we can create a postgis database, we'll call it *gis*.
```sh
sudo -u postgres -i
createuser username
createdb -E UTF8 -O username gis
```


Obviously you should replace *username* with a user of your choice. We will need that *user* later on.
Just in case you are lazy:
```sh
sudo useradd -m username
sudo passwd username
```


The next command should put you at a *"postgres=#"* prompt. This way we can setup the database for our intended purpose.
```sh
psql
```
Next we will connect to the database *gis*.
```sh
\c gis
```
**Response:** 'You are now connected to databse *gis* as user *postgres*.'
```sh
CREATE EXTENSION postgis;
```
**Response:** 'CREATE EXTENSION'
```sh
CREATE EXTENSION hstore;
```
**Response:** 'CREATE EXTENSION'
```sh
ALTER TABLE geometry_columns OWNER TO username;
```
**Response:** 'ALTER TABLE'
```sh
ALTER TABLE spatial_ref_sys OWNER TO username;
```
**Response:** 'ALTER TABLE'

We are done here, now we can go back to our Linux prompt and logout of the *postgres* user.
```sh
\q
exit
```

## osm2pgSQL
[osm2pgsql](https://github.com/openstreetmap/osm2pgsql) is used to load *OpenStreetMap data* into our *postgis* database.

### **From now on we'll be working on our created user.**

Just install it:
```sh
sudo apt-get install osm2pgsql
```

## Mapnik
Mapnik is a open source toolkit for rendering maps, written in C++. It is used to render the four main *Slippy Map* layers on the OpenstreetMap website.

Same procedure, prerequisites, dependencies and installation.
```sh
sudo apt-get install autoconf apache2-dev libtool libxml2-dev libbz2-dev libgeos-dev libgeos++-dev libproj-dev gdal-bin libmapnik-dev mapnik-utils python-mapnik
```
If there is an response in the end, you probably did something wrong.
```sh
python
>>> import mapnik
>>>
```
We should be good, let's continue.
```sh
>>> quit ()
```

## Renderd
*mod_tile* is a module for apache that will handle our tile requests. *renderd* is actually just a daemon, that ... renders tiles. 

*mod_tile* originates from [here](https://github.com/openstreetmap/mod_tile), 
alternatively [this one](https://github.com/SomeoneElseOSM/mod_tile) can be used, which supposedly supports Ubuntu 16.04, but we don't care about that.

In our case we just install it, it is included/a part of mod_tile so here we go:
```sh
sudo apt-get install libapache2-mod-tile
```

## Stylesheet
We pretty much installed everything thats necessary, but now we will have to download and configure a stylesheet.

The style we'll use here is the one that is used by the *default map* on [openstreetmap.org](ttps://www.openstreetmap.org/#map=3/19.97/10.11). OpenStreetMap Carto can be found [here](https://github.com/gravitystorm/openstreetmap-carto/).

We will store the style-sheet in the `src`-folder in the home directory of our *user*.
```sh
mkdir ~/src
cd ~/src
git clone git://github.com/gravitystorm/openstreetmap-carto.git
cd openstreetmap-carto
```
Next, we'll install a suitable version of the *carto* compiler.
```sh
sudo apt install npm nodejs
sudo npm install -g carto
carto -v 
```
**Response:** `carto 1.1.0` or higher.

Now we convert the carto project *project.mml* into a Mapnik XML stylesheet *mapnik.xml*.
```sh
carto project.mml > mapnik.xml
```
You should have a Mapnik XML stylesheet at `/home/username/src/openstreetmap-carto/mapnik.xml` now.

## Data
Intially, we'll only load a small amount of test data. We'll use *montenegro*, which is currently about *20 MB* in size. 
http://download.geofabrik.de/europe/montenegro-latest.osm.pbf

```sh
mkdir ~/data
cd ~/data
wget http://download.geofabrik.de/europe/montenegro-latest.osm.pbf
```
The following command will insert the OpenStreetMap data into the database. This might take a while, especially If you intend to insert the entire planet. Additionally you might have to try some different `-C` values, based on your hardware.

`-C` Allocates 2.5 GB of memory to the import process.

```sh
osm2pgsql -d gis --create --slim  -G --hstore --tag-transform-script ~/src/openstreetmap-carto/openstreetmap-carto.lua -C 2500 --number-processes 1 -S ~/src/openstreetmap-carto/openstreetmap-carto.style ~/data/montenegro-latest.osm.pbf
```

**Response:** 'Osm2pgsql took `?`s overall'

## Shapefile
Although most of the data used to create the map is directly from the OpenStreetMap data file that you downloaded above, some shapefiles for things like low-zoom country bondaries are still needed. To download and index these:
```sh
cd ~/src/openstreetmap-carto/
scripts/get-shapefiles.py
```
**Response:** '... script complete.'

## Fonts
The names used for countries and places around the world aren't all written in latin characters, to make sure everything works as intended we should download those.
```sh
sudo apt-get install fonts-noto-cjk fonts-noto-hinted fonts-noto-unhinted ttf-unifont
```

## Configuration
The config file is located here `/usr/local/etc/renderd.conf`. Obviously you can use any text editor, we'll use vim.
```sh
sudo vim /etc/renderd.conf
```
```sh
[renderd]
stats_file=/var/run/renderd/renderd.stats
socketname=/var/run/renderd/renderd.sock
num_threads=4
tile_dir=/var/lib/mod_tile


[mapnik]
plugins_dir=/usr/lib/mapnik/3.0/input
font_dir=/usr/share/fonts/
font_dir_recurse=1

[default]
URI=/hot/
XML=/home/username/src/openstreetmap-carto/mapnik.xml
TILEDIR=/var/lib/mod_tile
;HOST=tile.openstreetmap.org
;SERVER_ALIAS=http://a.tile.openstreetmap.org
;SERVER_ALIAS=http://b.tile.openstreetmap.org
;HTCPHOST=proxy.openstreetmap.org
TILESIZE=256 
```
Lines that might need changing:
```sh
num_threads=4
```
This should be *below* or *equal* to your memory. So if you have 2 GB of memory, you'll want 2 or lower.
```sh
XML=/home/username/src/openstreetmap-carto/mapnik.xml
```
Lines like this one might need to be changed If you decided to use a different directory.

You can find out your mapnik plugin directory with the following command:
```sh
mapnik-config --input-plugins
```
## Apache
```sh
sudo mkdir /var/lib/mod_tile
sudo chown username /var/lib/mod_tile
sudo mkdir /var/run/renderd
sudo chown username /var/run/renderd
```
Get *mod_tile* going:
```sh
sudo vim /etc/apache2/conf-available/mod_tile.conf
```
Add this line:
```sh
LoadModule tile_module /usr/lib/apache2/modules/mod_tile.so
```
Afterwards you'll want to do this:
```sh
sudo a2enconf mod_tile
```
Get *renderd* going:
```sh
sudo vim /etc/apache2/sites-available/000-default.conf
```
Add this between *"ServerAdmin"* & *"DocumentRoot"*:
```sh
LoadTileConfigFile /etc/renderd.conf
ModTileRenderdSocketName /var/run/renderd/renderd.sock
# Timeout before giving up for a tile to be rendered
ModTileRequestTimeout 0
# Timeout before giving up for a tile to be rendered that is otherwise missing
ModTileMissingRequestTimeout 30
```
And reload your apache **twice**:
```sh
sudo service apache2 reload
sudo service apache2 reload
```
## Run
With this command you can renderd files manually.
```sh
renderd -f -c /etc/renderd.conf
```
Now you can access your very own tile server via http://localhost/hot/0/0/0.png
The prefered way would obviously be a service, which is setup already.

## Service
A service should already be configured. Just restart the service and you'll be ready to go.
```sh
systemctl restart renderd
```

## Use with icinga2-module-map
If you want to use this vagrant box for your icinga-2-module-map, you just have to add this: 
`http://localhost/hot/{z}/{x}/{y}.png` 

Just login to your **web-interface**. Navigate down to **configuration** and choose **modules**. You should find **map** somewhere here, If you installed the maps-module correctly beforehand. Change from the **module:map**-tab to the **configuration**-tab and add *http://localhost/hot/{z}/{x}/{y}.png* to **URL for tile server**, which should be empty by default - so it uses the *openstreetmaps-servers*

## Adding
So there is no easy way to add a country. You have to merge the *osm.pbf*-files together and import them into the database.

## Merging
You might want to use just a specific region plus some specific countries. To do just that we'll merge them, for example USA and Great Britain. We will do that by using "Osmium", which is used to convert and process OpenStreetMapFiles.

Lets say we've downloaded Great Britain already and its in our database aswell.
download.geofabrik.de/europe/great-britain-latest.osm.pbf
Northern America is next.
download.geofabrik.de/north-america-latest.osm.pbf

[Osmium](https://wiki.openstreetmap.org/wiki/Osmium) can be found [here](https://osmcode.org/osmium-tool/).

With both files at ~/data/ we will issue the following command:
```sh
osmium merge north-america-latest.osm.pbf great-britain-latest.osm.pbf -o north-america_great_britain-latest.osm.pbf  
```
Syntax:
```sh
osmium merge file1.osm file2.osm -o merged.osm
```
Now we just need to load the file into our database
```sh
osm2pgsql --create ... map_with_new_region.pbf
```
Another tool that provides the same feature is [osmosis](https://wiki.openstreetmap.org/wiki/Osmosis), you can find it [here](https://www.osmosis.org/).

## Other
- [Maintaining & Updating](https://wiki.openstreetmap.org/wiki/User:SomeoneElse/Ubuntu_1804_tileserver_load#Updating_your_database_as_people_edit_OpenStreetMap)

