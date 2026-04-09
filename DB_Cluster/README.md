Database cluster is basically a group of databases managed by the server postgres
a command initdb is used to create a cluster , if ur using docker containers then u might add initdb to be executed automatically when running the postgres container as the entrypoint , 3 databases are created by default template1 , template2 and postgres

PG_DATA is the env variable that stores the path to the cluster data directory

under this directory there is : 

a directory BASE which contains subdirectories representing each a database 
some config files
other directories like :
global which contains  tables that store metadata about the cluster like pg_database
the folders names are the oids of the databases/tables/indexes 

what is an oid? :
it's a number assigned uniquely to a database object (table,database,indexe ect)
to search oids we use the sql command 
SELECT datname,oid from pg_database 

exemples des fichiers importants qu'on peut trouver en PGDATA :
**PG_VERSION** Le fichier PG_VERSION définit la version de tout le cluster. Toutes les bases de données (instances) situées sous le dossier base/ doivent impérativement respecter cette version.
Le fichier current_logfiles est un fichier temporaire généré par PostgreSQL uniquement lorsque le collecteur de journaux (logging collector) est activé.

**current_logfiles** Il sert de pointeur en temps réel. Il indique quel fichier de log est actuellement utilisé par le serveur pour écrire les erreurs, les requêtes lentes ou les logs de connexion.