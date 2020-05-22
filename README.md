# Nominatim Docker (Nominatim version 3.4)
Nominatim (from the Latin, 'by name') is a tool to search OpenStreetMap data by name and address (geocoding) and to generate synthetic addresses of OSM points (reverse geocoding). An instance with up-to-date data can be found at https://nominatim.openstreetmap.org. Nominatim is also used as one of the sources for the Search box on the OpenStreetMap home page.

## General Build/Setup
The setup and build steps below are taken from the project [https://github.com/mediagis/nominatim-docker](https://github.com/mediagis/nominatim-docker).
We are using the latest version of nominatim, v3.4. The nominatim project can be found [here](https://github.com/osm-search/Nominatim) and the documentation is found [here](https://nominatim.org/release-docs/develop/). You can follow these steps to get things running on your local machine for testing/dev.

1. Build the image

   ```
   docker build -t nominatim .
   ```

1. Copy <your_country>.osm.pbf to a local directory (i.e. /home/me/nominatimdata). The osm.pbf files can be obtained from
   geofabrik's website at [https://download.geofabrik.de/index.html](https://download.geofabrik.de/index.html). For general
   dev purposes, the file for Monaco is typically used as it is very small. The data loading process can be very time consuming.
   It takes ~1.5-2 hrs just to load Texas into the database, depending on your system's specifications.

1. Initialize Nominatim Database

   ```
   docker run -t -v /home/me/nominatimdata:/data nominatim  sh /app/init.sh /data/<your_country>.osm.pbf postgresdata 4
   ```
   Where 4 is the number of threads to use during import. In general the import of data in postgres is a very time consuming
   process that may take hours or days. If you run this process on a multiprocessor system make sure that it makes the best use
   of it. You can delete the /home/me/nominatimdata/<your_country>.osm.pbf once the import is finished.


1. After the import is finished the /home/me/nominatimdata/postgresdata folder will contain the full postgres binaries of
   a postgis/nominatim database. The easiest way to start nominatim as a single node is the following:

   ```
   docker run --restart=always -p 6432:5432 -p 7070:8080 -d --name nominatim -v /home/me/nominatimdata/postgresdata:/var/lib/postgresql/11/main nominatim bash /app/start.sh
   ```

1. Advanced configuration. If necessary you can split the osm installation into a database and restservice layer

   In order to set the nominatib-db only node:

   ```
   docker run --restart=always -p 6432:5432 -d -v /home/me/nominatimdata/postgresdata:/var/lib/postgresql/11/main nominatim sh /app/startpostgres.sh
   ```
   After doing this create the /home/me/nominatimdata/conf folder and copy there the docker/local.php file. Then uncomment the following line:

   ```
   @define('CONST_Database_DSN', 'pgsql://nominatim:password1234@192.168.1.128:6432/nominatim'); // <driver>://<username>:<password>@<host>:<port>/<database>
   ```

   You can start the nominatim-rest only node with the following command:

   ```
   docker run --restart=always -p 7070:8080 -d -v /home/me/nominatimdata/conf:/data nominatim sh /app/startapache.sh
   ```

1. Configure incremental update. By default CONST_Replication_Url in local.php has been configured for Texas.
   If you want a different update source, you will need to declare `CONST_Replication_Url` in local.php. Documentation [here](https://github.com/openstreetmap/Nominatim/blob/master/docs/admin/Import-and-Update.md#updates). For example, to use the daily country extracts diffs for Monaco from geofabrik add the following:

   ```
   @define('CONST_Replication_Url', 'https://download.geofabrik.de/europe/monaco-updates');
   ```

  Now you will have a fully functioning nominatim instance available at : [http://localhost:7070/](http://localhost:7070).


## Updating Nominatim DB

Full documentation for Nominatim updates available [here](https://github.com/openstreetmap/Nominatim/blob/master/docs/admin/Import-and-Update.md#updates). For a list of other methods see the output of:

  ```
  docker exec -it nominatim sudo -u postgres ./src/build/utils/update.php --help
  ```

The following command must be run once to set up the updates process before the updates can begin:
  ```
  docker exec -it nominatim sudo -u postgres ./src/build/utils/update.php --init-updates
  ``` 

The following command will update the nominatim database with the most recent osm changes:

  ```
  docker exec -it nominatim sudo -u postgres ./src/build/utils/update.php --import-osmosis
  ```

The following command will keep your database constantly up to date:

  ```
  docker exec -it nominatim sudo -u postgres ./src/build/utils/update.php --import-osmosis-all
  ```
## Production Build/Setup

The production setup uses the build steps from above to build the image and run the container on an ec2. Future versions will likely move the deployment to ecs with a persistent volume for the data directory. The following steps outline the exact procedure to set everything up on the server. The host must have git and docker installed prior to the build.

1. Connect to the host machine using ssh

1. Clone this repository to the host in the ec2-user's home directory

1. Create a local directory to store the osm extract and postgres data called nominatimdata at the same level as the project repo

   ```
   mkdir /home/ec2-user/nominatimdata
   ```

1. Navigate to the nominatimdata directory and copy texas-latest.osm.pbf to a local directory (i.e. /home/ec2-user/nominatimdata). The osm.pbf file can be obtained from
   geofabrik's website at [https://download.geofabrik.de/north-america/us/texas.html](https://download.geofabrik.de/north-america/us/texas.html).

   ```
   curl -O https://download.geofabrik.de/north-america/us/texas-latest.osm.pbf
   ```
   
1. Build the image

   ```
   docker build -t nominatim .
   ```

1. Initialize Nominatim Database

   ```
   docker run -t -v /home/ec2-user/nominatimdata:/data nominatim  sh /app/init.sh /data/texas-latest.osm.pbf postgresdata 2
   ```
   Where 2 is the number of threads to use during import. We are building this on a machine with only 2 threads available. In general the import of data in postgres is a very time consuming process, it should take ~1.5 hrs to load Texas.

1. After the import is finished the /home/ec2-user/nominatimdata/postgresdata folder will contain the full postgres binaries of a postgis/nominatim database. The   
   easiest way to start nominatim as a single node is the following:

   ```
   docker run --restart=always -p 6432:5432 -p 80:8080 -d --name nominatim -v /home/ec2-user/nominatimdata/postgresdata:/var/lib/postgresql/11/main nominatim bash /app/start.sh
   ```