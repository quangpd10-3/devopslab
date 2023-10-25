#login to masternode as "postgres" user 
sudo -u postgres psql

#create a replication user that will be used to initiate the replication process from the primary node.
CREATE ROLE replica_user WITH REPLICATION LOGIN PASSWORD 'P@ssword321';

#log out from the PostgreSQL prompt
postgres-# \q

#edit configuration file : /etc/postgresql/16/main/postgresql.conf

listen_addresses= '10.15.0.184' #server’s IP address
wal_level = logical
wal_log_hints = on

#save and exit

#append this two lines to configuration file to allows the replica ( 10.15.0.185 and 10.15.0.186 )
#to connect with the master node using the replica_user.
host  replication   replica_user  10.15.0.185/32   md5
host  replication   replica_user  10.15.0.186/32   md5

#save and exit and restart PostgresSQL
sudo systemctl restart postgresql

#Configure on Replica Nodes
#stop PostgreSQL service on the replica nodes
sudo systemctl stop postgresql

#remove all files in the replica’s data directory in order to start on a clean slate and make room for the primary node data directory.
sudo rm -rv /var/lib/postgresql/16/main/

#copy data from the primary node to the replica node.
sudo pg_basebackup -h 10.15.0.184 -U replica_user -X stream -C -S replica_1  -R -W -D /var/lib/postgresql/16/main/

#-X stream :instructs the pg_basebackup utility to stream and include the WAL files in the backup.
#-C -S replica_1 : allows create a replication slot before backup . -S option to specify the slot name is replica_1.
#-R : creates two files; an empty recovery configuration file called standby.signal and a primary node connection settings file called postgresql.auto.conf.
# The standby.signal file contains connection information about the primary node and the postgresql.auto.conf file informs your replica cluster that it should operate as a standby server.
#-W : request provide password for replica_user.
#-D :allows you to include the directory that you want to export the backup files in the replica nodes.

#grant ownership of the data directory to the postgres user.
sudo chown postgres -R /var/lib/postgresql/16/main/

#start again the PostgreSQL server. The replica will now be running in hot standby mode.
sudo systemctl start postgresql
