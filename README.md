# Nominatim Docker (Nominatim version 3.4)

1. Build
   ```
   docker build -t nominatim .
   ```
2. Copy <your_country>.osm.pbf to a local directory (i.e. /home/me/nominatimdata). The osm.pbf files can be obtained from
   geofabrik's website at: [https://download.geofabrik.de/index.html](https://download.geofabrik.de/index.html). For general
   dev purposes, the file for Monaco is typically used as it is very small. The data loading process can be very time consuming.
   It takes ~1.5-2 hrs just to load Texas into the database, depending on your system's specifications.

3. Initialize Nominatim Database
   ```
   docker run -t -v /home/me/nominatimdata:/data nominatim  sh /app/init.sh /data/<your_country>.osm.pbf postgresdata 4
   ```
   Where 4 is the number of threads to use during import. In general the import of data in postgres is a very time consuming
   process that may take hours or days. If you run this process on a multiprocessor system make sure that it makes the best use
   of it. You can delete the /home/me/nominatimdata/<your_country>.osm.pbf once the import is finished.


4. After the import is finished the /home/me/nominatimdata/postgresdata folder will contain the full postgress binaries of
   a postgis/nominatim database. The easiest way to start nominatim as a single node is the following:
   ```
   docker run --restart=always -p 6432:5432 -p 7070:8080 -d --name nominatim -v /home/me/nominatimdata/postgresdata:/var/lib/postgresql/11/main nominatim bash /app/start.sh
   ```

5. Advanced configuration. If necessary you can split the osm installation into a database and restservice layer

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

6. Configure incremental update. By default CONST_Replication_Url configured for Monaco.
   If you want a different update source, you will need to declare `CONST_Replication_Url` in local.php. Documentation [here](https://github.com/openstreetmap/Nominatim/blob/master/docs/admin/Import-and-Update.md#updates). For example, to use the daily regional extracts diffs for Texas from geofabrik add the following:
   ```
   @define('CONST_Replication_Url', 'https://download.geofabrik.de/north-america/us/texas-updates');
   ```

  Now you will have a fully functioning nominatim instance available at : [http://localhost:7070/](http://localhost:7070).


# Update

Full documentation for Nominatim update available [here](https://github.com/openstreetmap/Nominatim/blob/master/docs/admin/Import-and-Update.md#updates). For a list of other methods see the output of:
  ```
  docker exec -it nominatim sudo -u postgres ./src/build/utils/update.php --help
  ```

The following command will keep your database constantly up to date:
  ```
  docker exec -it nominatim sudo -u postgres ./src/build/utils/update.php --import-osmosis-all
  ```
