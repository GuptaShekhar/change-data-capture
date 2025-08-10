## Change Data Capture (CDC) with Debezium, MySQL, and Kafka
This repository provides a hands-on example of how to implement a real-time Change Data Capture (CDC) pipeline. It leverages Debezium, an open-source distributed platform, to monitor a **MySQL** database for row-level changes (INSERTs, UPDATEs, and DELETEs) and stream these events to **Apache Kafka**.

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

1. Clone this repository:
    ```
    git clone https://github.com/your-username/change-data-capture.git
    cd change-data-capture
    ```

2. Start the entire stack using Docker Compose:
    ```
    docker-compose up -d
    ```

3. Once the containers are running, you can connect to the MySQL database and start making changes to the tables.

4. Use a Kafka client to consume messages from the topics to see the change events in real time.

### Use Cases
This real-time CDC pipeline is ideal for:

- **Real-time Analytics**: Keeping data warehouses and other systems in sync for up-to-the-minute reporting.

- **Event-Driven Microservices**: Building reactive applications that respond to data changes in the source database.

- **Database Synchronization**: Replicating data to other databases or systems without complex batch jobs.
