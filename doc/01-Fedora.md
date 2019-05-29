# Manually building a tile server on Fedora
This page describes how to setup everything that is required to operate your own tile server.

Pretty much everything that's written here is based on the **[Installation Guide](https://github.com/anuragyayhdapu/Manual-Map-Tile-Server-Fedora)** by anuragyayhdapu. Other then that this has been a great help, even though that tutorial is on Ubuntu: **[switch2osm](https://switch2osm.org/manually-building-a-tile-server-18-04-lts/)** 

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

## Dependencies

This *[OSM](https://www.openstreetmap.org/#map=3/19.97/10.11) tile server* setup is build around five major parts: *[mod_tile](https://wiki.openstreetmap.org/wiki/Mod_tile); renderd; [mapnik](https://wiki.openstreetmap.org/wiki/Mapnik); [osm2pgsql](https://wiki.openstreetmap.org/wiki/Osm2pgsql); [postgis](https://wiki.openstreetmap.org/wiki/PostGIS)*.
I'll explain these as we get to it. In order to install all of these components, some dependencies have to be resolved first.

We'll have to configure rpm-fusion first:
```sh
sudo dnf install https://download1.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm 
sudo dnf install https://download1.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm
```

Now we get to the dependencies
```sh
sudo dnf update
sudo dnf groupinstall 'Development Tools'
sudo dnf install boost-devel autoconf libtool libxml2-devel geos geos-devel postgresql-devel bzip2-devel proj-devel munin-node munin protobuf-c-devel protobuf-c-compiler freetype-devel libpng12-devel libtiff-devel libicu-devel gdal-devel cairo-devel cairomm-devel httpd-devel git vim wget unifont unifont-fonts libgeotiff
```

## postgreSQL

*PostgreSQL* (or just *"Postgres"*) is an object-relational database, which we'll be using to store our *OSM data*. *Postgis* is an extension for *Postgres*. PostGIS adds additional geospatial types and functions which make handling spatial data in the database easier and more powerful. Basically it's perfect for us. Both are available pre-packaged on Fedora.
```sh
sudo dnf install postgresql postgresql-contrib postgis phpPgAdmin
```
Followed by initiating and enabling
```
sudo systemctl enable postgresql
sudo postgresql-setup --initdb --unit postgresql
sudo systemctl start postgresql
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

Add these to make sure we meet all prerequisites:
```sh
sudo dnf install cmake gcc-c++ lua-devel
```

### **From now on we'll be working on our created user.**
```sh
mkdir ~/src
cd ~/src
git clone git://github.com/openstreetmap/osm2pgsql.git
cd osm2pgsql
```
Now we can actually get to osm2pgsql.
```sh
mkdir build && cd build
cmake ..
```
**Response:** '-- Build files have been written to: */home/username/src/osm2pgsql/build*'
```sh
make
```
**Response:** '[100%] Built target *osm2pgsql*'
```sh
sudo make install
```

## Mapnik
Mapnik is a open source toolkit for rendering maps, written in C++. It is used to render the four main *Slippy Map* layers on the OpenstreetMap website.

Same procedure, prerequisites, dependencies and installation.

```sh
sudo dnf install gdalcpp-devel mapnik-devel mapnik-utils python-mapnik fribidi-devel libtool-ltdl-devel python2-devel libpqxx-devel
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
Just add the copr-repo:
```sh
dnf copr enable tartare/mod_tile 
```
And install it like this:
```sh
???????????????????????
```
## Stylesheet
We pretty much installed everything thats necessary, but now we will have to download and configure a stylesheet.

The style we'll use here is the one that is used by the *default map* on [openstreetmap.org](ttps://www.openstreetmap.org/#map=3/19.97/10.11). OpenStreetMap Carto can be found [here](https://github.com/gravitystorm/openstreetmap-carto/).

We will store the style-sheet in the `src`-folder in the home directory of our *user*.
```sh
cd ~/src
git clone git://github.com/gravitystorm/openstreetmap-carto.git
cd openstreetmap-carto
```
Next, we'll install a suitable version of the *carto* compiler.
```sh
sudo dnf install npm nodejs
sudo npm config delete proxy
sudo npm install -g carto
carto -v
```
**Response:** `carto 1.1.0`
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
sudo dnf install google-noto-cjk-fonts ttfautohint
```

## Configuration
The config file is located here `/etc/renderd.conf`. Obviously you can use any text editor, we'll use vim.
```sh
sudo vim /etc/renderd.conf
```
```sh
[renderd]
num_threads=4
tile_dir=/var/lib/mod_tile
stats_file=/var/run/renderd/renderd.stats

[mapnik]
plugins_dir=paste plugins directory here
font_dir=/usr/share/fonts
font_dir_recurse=1

[default]
URI=/osm_tiles/
TILEDIR=/var/lib/mod_tile
XML=/home/renderaccount/src/openstreetmap-carto/mapnik.xml
HOST=tile.openstreetmap.org
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

You can find out your mapnik plugin directory with the follwing command:
```sh
mapnik-config --input-plugins
```

## Apache
```sh
sudo systemctl enable httpd
sudo systemctl start httpd
sudo systemctl reload httpd
```
We have to disable SELinux temporarily
```sh
sudo setenforce 0
```

## Run
With this command you can renderd files manually.
```sh
renderd -f
```
Now you can access your very own tile server via http://localhost/osm_tiles/0/0/0.png
We will change this into a service in the next step.

## Service
Now we'll make this a service instead.
First of change renderd.init *RUNASUSER* to our *tile* user and follow the steps below.
```sh
sudo vim /etc/init.d/renderd
```
Paste the following and adjust as you see fit.
```sh
#! /bin/sh
### BEGIN INIT INFO
# Provides:          renderd
# Required-Start:    $remote_fs
# Required-Stop:     $remote_fs
# Should-Start:      postgresql
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
# Short-Description: Mapnik rendering daemon
# Description:       Mapnik rendering daemon.
### END INIT INFO

# Do NOT "set -e"

# PATH should only include /usr/* if it runs after the mountnfs.sh script
PATH=/sbin:/usr/sbin:/bin:/usr/bin
DESC="Mapnik rendering daemon"
NAME=renderd
DAEMON=/usr/local/bin/$NAME
DAEMON_ARGS="-c /usr/local/etc/renderd.conf"
PIDSOCKDIR=/var/run/$NAME
PIDFILE=$PIDSOCKDIR/$NAME.pid
SCRIPTNAME=/etc/init.d/$NAME
RUNASUSER=username
```
Make sure its executable for your user:
```sh
sudo chmod u+x /etc/init.d/renderd
```
Start & enable the service:
```sh
sudo /etc/init.d/renderd start
sudo systemctl enable renderd
```
## Use with icinga2-module-map
If you want to use this vagrant box for your icinga-2-module-map, you just have to add this: 
`http://localhost/osm_tiles/{z}/{x}/{y}.png` 

Just login to your **web-interface**. Navigate down to **configuration** and choose **modules**. You should find **map** somewhere here, If you installed the maps-module correctly beforehand. Change from the **module:map**-tab to the **configuration**-tab and add *http://localhost/osm_tiles/{z}/{x}/{y}.png* to **URL for tile server**, which should be empty by default - so it uses the *openstreetmaps-servers*

