# Instructions to create a Neo4j Container with data from dump file

# Step 1: Create container
`docker run   --name=olbapd-neo4j  --publish=7474:7474 --publish=7473:7473 --publish=7687:7687 -it --env=NEO4J_ACCEPT_LICENSE_AGREEMENT=yes --env=NEO4J_AUTH=neo4j/admin neo4j:enterprise`

We name the container `olbapd-neo4j` enable port 7474, 7687 and 7473. We must use the enterprise version in order to be able to do access the neo4j-admin. The enterprise version is free on development mode. In order to get a license one must pay or request an academic license.

# Step 2: Stopping the database

Open the cypher shell and access the system database.

`docker exec -it olbapd-neo4j cypher-shell -u neo4j -p admin -d system 'STOP DATABASE neo4j'`
Then stop the `neo4j` database or the default one being used with the following command:

# Step 3: Copy the dump file into the container

Change the path to match your dump file with the database to load.

`docker cp C:\Users\pablo\Downloads\neo4j-20210316.dump olbapd-neo4j:/var/lib/neo4j`

# Step 4: Create the database from the dump file

`docker exec -it olbapd-neo4j neo4j-admin load --verbose --from=/var/lib/neo4j/neo4j-20210316.dump --force --database=graph.db`

# Step 5: Start the database

`docker exec -it olbapd-neo4j cypher-shell -u neo4j -p admin -d system 'START DATABASE neo4j'`

List the databases using `SHOW DATABASES` to make sure the database is up and running. 

If the database is not running a restart of the container may be requiered.

# References
https://markhneedham.com/blog/2020/01/28/neo4j-database-dump-docker-container/

