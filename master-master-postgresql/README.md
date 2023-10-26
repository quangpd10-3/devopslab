#Edit /etc/hosts to allow specific network subnet can access
Hostname      IP Address
---------------------------
postgres01    10.15.0.184
postgres02    10.15.0.185

#create a new database and user that will be used for the Bucardo installation.
#also create a new database with the schema for testing the PostgreSQL replication.
#run command on both server:
cd /var/lib/postgresql
sudo -u postgres psql

CREATE USER bucardo WITH SUPERUSER;
CREATE DATABASE bucardo OWNER bucardo;

#After creating the database and user for Bucardo
#next create a new database for testing the replication on PostgreSQL server.

CREATE DATABASE testdb;
\c testdb;

#create a new table 'users'
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  first_name VARCHAR(255),
  last_name VARCHAR(255) NOT NULL,
  city VARCHAR(255)
);

#set up both PostgreSQL servers to run on an internal IP address.
#Also, set up the PostgreSQL authentication to allow connections between servers as trusted to PostgreSQL.
sudo nano /etc/postgresql/16/main/postgresql.conf
#set up PostgreSQL to be running on an internal IP address on each server.
listen_addresses = 'localhost,10.15.0.184'
#do the same thing on server2
listen_addresses = 'localhost,10.15.0.185'


#open the default PostgreSQL authentication config file '/etc/postgresql/16/main/pg_hba.conf'
#With this, any local connections and connections from user bucardo will be trusted.
#Also, connections of users postgres and bucardo that comes from the postgres02 are trusted.
#add this on server1
local    all             all                                    trust
local    all             bucardo                                trust
host    all             postgres         10.15.0.185/24       trust
host    all             bucardo          10.15.0.185/24       trust

#add this on server2
local    all             all                                    trust
local    all             bucardo                                trust
host    all             postgres         10.15.0.184/24       trust
host    all             bucardo          10.15.0.184/24       trust

#restart on both server 
sudo systemctl restart postgresql

#install Burcado
#Setup Multi-Master Replication with Bucardo only on server1
#create a new data and log directory for Bucardo.

sudo mkdir -p /var/run/bucardo /var/log/bucardo
touch /var/log/bucardo/log.bucardo

bucardo install #input P to proceed

#define the database server and database name that will be replication.
#That information will be stored as the 'server1' for the PostgreSQL server ppstgres01 and 'server2' for the PostgreSQL server postgres02.
bucardo add database server1 dbname=testdb
bucardo add database server2 dbname=testdb host=10.15.0.185

#Add the table schema that want to replicate. 
bucardo add table public.users db=server1
bucardo add table public.users db=server2

#add all tables on the database and also create a relgroup .
bucardo add all tables --her=testdbSrv1 db=server1
bucardo add all tables --her=testdbSrv2 db=server2

#With the relgroup and table added, start the synchronization process for both PostgreSQL servers.
#sync 'testdbSrv1' will sync 'server1' and 'server2'. 
#sync 'testdbSrv2' will sync 'server2' and 'server1'.
bucardo add sync testdbSrv1 relgroup=testdbSrv1 db=server1,server2
bucardo add sync testdbSrv2 relgroup=testdbSrv2 db=server2,server1

bucardo restart sync

#now should receive an output : Both sync 'testdbSrv1' and 'testdbSrv2' on the state 'Good' which confirms sucess.
bucardo status









