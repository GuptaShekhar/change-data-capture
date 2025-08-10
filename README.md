## Change Data Capture (CDC) with Debezium, MySQL, and Kafka
This repository provides a hands-on example of how to implement a real-time Change Data Capture (CDC) pipeline. It leverages Debezium, an open-source distributed platform, to monitor a **MySQL** database for row-level changes (INSERTs, UPDATEs, and DELETEs) and stream these events to **Apache Kafka**.

### Use Cases
This real-time CDC pipeline is ideal for:

- **Real-time Analytics**: Keeping data warehouses and other systems in sync for up-to-the-minute reporting.

- **Event-Driven Microservices**: Building reactive applications that respond to data changes in the source database.

- **Database Synchronization**: Replicating data to other databases or systems without complex batch jobs.

### How It Works
The project uses a Dockerized environment to set up all the necessary components:

- A **MySQL** database with binary logging enabled, which is essential for Debezium to read the transaction log.

- **Apache Kafka** and **ZooKeeper** to act as the message broker for the change events.

- **Kafka Connect** with the Debezium MySQL Connector, which is configured to connect to the database, read the binlog, and publish change events to Kafka topics.

The data changes are captured in near real-time, providing a robust and scalable solution for various use cases.

### Getting Started
To run this project locally, you will need to have Docker and Docker Compose installed on your machine.

**Prerequisites**

- Docker

- Docker Compose

**Running the Stack**


1. **Configure MySQL for Debezium**

    Before you can capture changes, your MySQL instance must be configured with binary logging enabled.

    A. **Install and Configure MySQL 8.0**

    Install MySQL 8.0 using Homebrew on a Mac and then navigate to the configuration directory to create a my.cnf file.
    ```
    brew install mysql@8.0
    cd /opt/homebrew/var/mysql
    touch my.cnf
    vim my.cnf
    ```
    Add the following content to my.cnf to enable binary logging and GTID consistency, which are required for Debezium.
    ```
    [mysqld]
    server_id=223344
    log_bin=mysql-bin
    binlog_format=ROW
    gtid_mode=ON
    enforce_gtid_consistency=ON
    binlog_row_image=FULL
    bind-address = 0.0.0.0
    ```
    B. **Create a Debezium User**

    Create a dedicated user for Debezium with the necessary permissions to read the binlog.
    ```
    CREATE USER 'debezium'@'%' IDENTIFIED BY 'dbz';
    GRANT SELECT, RELOAD, SHOW DATABASES, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'debezium'@'%';
    FLUSH PRIVILEGES;
    ```

2. **Run the Docker Stack**

    This step involves cloning the repository and starting the services using Docker Compose.

    A. **Clone the Repository**

    Clone the repository and navigate into the directory. Alternatively, you can create a project with the necessary docker-compose.yml and debezium-connector-config.json files.
    ```
    git clone https://github.com/your-username/change-data-capture.git
    cd change-data-capture
    ```
    B. **Start the Services**

    Use Docker Compose to start MySQL, Kafka, and Kafka Connect in the background.
    ```
    docker-compose up -d
    ```
3. **Establish the Debezium Connector**

    Now that all services are running, you need to configure the Debezium connector to start capturing changes from MySQL and sending them to Kafka.

    A. **Check for Existing Connectors**

    Initially, there should be no active connectors.
    ```
    curl -H "Accept:application/json" localhost:8083/connectors/
    ```
    Expected output:
    ```
    []
    ```
    B. **Create the Connector**

    Submit the debezium-connector-config.json file to the Kafka Connect API to establish the connection.
    ```
    curl -X POST -H "Content-Type: application/json" \
    --data @debezium-connector-config.json \
    http://localhost:8083/connectors
    ```
    Expected output:
    ```
    {"name":"mysql-connector","config":{"connector.class":"io.debezium.connector.mysql.MySqlConnector","tasks.max":"1","database.hostname":"host.docker.internal","database.port":"3306","database.user":"debezium","database.password":"dbz","database.server.id":"223344","topic.prefix":"mysql_test_server","database.include.list":"test","table.include.list":"test.user","schema.history.internal.kafka.bootstrap.servers":"kafka:9092","schema.history.internal.kafka.topic":"schema-changes.mysql_test_server","database.allowPublicKeyRetrieval":"true","database.ssl.mode":"disabled","snapshot.locking.mode":"none","name":"mysql-connector"},"tasks":[],"type":"source"}
    ```
    C. **Verify Connector Status**

    Check the status to ensure the connector is running successfully.
    ```
    curl -s http://localhost:8083/connectors/mysql-connector/status
    ```
    Expected output:
    ```
    {"name":"mysql-connector","connector":{"state":"RUNNING","worker_id":"172.19.0.4:8083"},"tasks":[{"id":0,"state":"RUNNING","worker_id":"172.19.0.4:8083"}],"type":"source"}
    ```
    D. **Connector Management**

    If you need to update the connector configuration or if the MySQL IP address changes, you can delete and re-create the connector.
    ```
    curl -X DELETE http://localhost:8083/connectors/mysql-connector

    curl -X POST -H "Accept:application/json" -H "Content-Type:application/json"
    --data @debezium-connector-config.json http://localhost:8083/connectors
    ```
    If MySQL itself needs to be restarted due to configuration changes:
    ```
    brew services restart mysql@8.0
    docker-compose down
    docker-compose up -d
    ```
4. **Test the Pipeline**

    Now, you can make a change in the MySQL database and verify that it is captured by Kafka.

    A. **Insert Data into MySQL**
    
    Connect to your MySQL database and insert a new row into the user table.
    ```
    INSERT INTO user (email, first_name, last_name, date_of_birth)
    VALUES
        ('john.doe@example.com', 'John', 'Doe', '1990-05-15');
    ```
    B. **Consume Messages from Kafka**
    Navigate into the Kafka container to list the topics and then consume messages from the topic that corresponds to your user table.
    ```
    docker exec -it <container_id> bash
    ```
    Replace <container_id> with the ID of your Kafka container.

    List the topics:
    ```
    kafka-topics --list --bootstrap-server localhost:9092
    ```
    Expected output:
    ```
    __consumer_offsets
    my_connect_configs
    my_connect_offsets
    my_connect_statuses
    mysql_test_server
    mysql_test_server.test.user
    schema-changes.mysql_test_server
    ```
    Consume the messages from the mysql_test_server.test.user topic:
    ```
    kafka-console-consumer --bootstrap-server localhost:9092 --topic mysql_test_server.test.user --from-beginning
    ```
    Expected output:
    ```
    {"schema":{"type":"struct","fields":[{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"},{"type":"string","optional":false,"field":"email"},{"type":"string","optional":false,"field":"first_name"},{"type":"string","optional":false,"field":"last_name"},{"type":"int32","optional":true,"name":"io.debezium.time.Date","version":1,"field":"date_of_birth"}],"optional":true,"name":"mysql_test_server.test.user.Value","field":"before"},{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"},{"type":"string","optional":false,"field":"email"},{"type":"string","optional":false,"field":"first_name"},{"type":"string","optional":false,"field":"last_name"},{"type":"int32","optional":true,"name":"io.debezium.time.Date","version":1,"field":"date_of_birth"}],"optional":true,"name":"mysql_test_server.test.user.Value","field":"after"},{"type":"struct","fields":[{"type":"string","optional":false,"field":"version"},{"type":"string","optional":false,"field":"connector"},{"type":"string","optional":false,"field":"name"},{"type":"int64","optional":false,"field":"ts_ms"},{"type":"string","optional":true,"name":"io.debezium.data.Enum","version":1,"parameters":{"allowed":"true,last,false,incremental"},"default":"false","field":"snapshot"},{"type":"string","optional":false,"field":"db"},{"type":"string","optional":true,"field":"sequence"},{"type":"string","optional":true,"field":"table"},{"type":"int64","optional":false,"field":"server_id"},{"type":"string","optional":true,"field":"gtid"},{"type":"string","optional":true,"field":"file"},{"type":"int64","optional":false,"field":"pos"},{"type":"int32","optional":false,"field":"row"},{"type":"int64","optional":true,"field":"thread"},{"type":"string","optional":true,"field":"query"}],"optional":false,"name":"io.debezium.connector.mysql.Source","field":"source"},{"type":"string","optional":false,"field":"op"},{"type":"int64","optional":true,"field":"ts_ms"},{"type":"struct","fields":[{"type":"string","optional":false,"field":"id"},{"type":"int64","optional":false,"field":"total_order"},{"type":"int64","optional":false,"field":"data_collection_order"}],"optional":true,"name":"event.block","version":1,"field":"transaction"}],"optional":false,"name":"mysql_test_server.test.user.Envelope","version":1},"payload":{"before":null,"after":{"id":1,"email":"john.doe@example.com","first_name":"John","last_name":"Doe","date_of_birth":7439},"source":{"version":"2.2.1.Final","connector":"mysql","name":"mysql_test_server","ts_ms":1754821834000,"snapshot":"last","db":"test","sequence":null,"table":"user","server_id":0,"gtid":null,"file":"binlog.000003","pos":157,"row":0,"thread":null,"query":null},"op":"r","ts_ms":1754821834315,"transaction":null}}
    ```