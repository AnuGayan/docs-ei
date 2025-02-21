# Siddhi Query API

## Introduction

Siddhi query API is the REST API exposed by the Streaming Integrator (SI). It gives you a set of APIs to perform all of the essential operations; regarding to developing, testing and querying of Streaming Integration applications.

Siddhi query API provides APIs related to:
- Siddhi application management (such as creating, updating, deleting a Siddhi application; Listing all running Siddhi applications etc.)
- Event simulation
- Authentication and Permission management
- Health check
- Siddhi Store operations

Refer [Streaming Integration REST API Guide](https://ei.docs.wso2.com/en/next/streaming-integrator/ref/si-rest-api-guide/) for a comprehensive reference on the Siddhi query API.

Using simple examples, this tutorial demonstrates how you can use the Siddhi query API to perform essential operations that you would need to perform using the SI.

## Tutorial Outline

- [Preparing the server](#preparing-the-server)
- [Creating a Siddhi application](#Creating-a-Siddhi-application)
- [Running a Siddhi Store API query](#Running-a-Siddhi-Store-API-query)
- [Fetching the status of a Siddhi Application](#Fetching-the-status-of-a-Siddhi-Application)
- [Taking a snapshot of a Siddhi Application](#Taking-a-snapshot-of-a-Siddhi-Application) 
- [Restoring a Siddhi Application via a snapshot](#Restoring-a-Siddhi-Application-via-a-snapshot)

## Preparing the server

!!!Prerequisites
    Before you begin:<br/>
    - You need to have access to a MySQL instance.<br/>
    - Add the MySQL JDBC driver into the `<SI_HOME>/lib` directory as follows:
      1. Download the MySQL JDBC driver from [the MySQL site](https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-5.1.45.tar.gz).
      2. Unzip the archive.
      3. Copy the `mysql-connector-java-5.1.45-bin.jar` to the `<SI_HOME>/lib` directory.
      4. Start the SI server.

1. Let's create a new database in the MySQL server which you are to use throughout this tutorial. To do this, execute the following query.
    ```
    CREATE SCHEMA production;
    ```
2. Create a new user by executing the following SQL query.
    ```
    GRANT SELECT, RELOAD, SHOW DATABASES, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'wso2si' IDENTIFIED BY 'wso2';
    ```
3. Switch to the `production` database and create a new table, by executing the following queries:
    ```
    use production;
    ```
    ```
    CREATE TABLE SweetProductionTable (name VARCHAR(20),amount double(10,2));
    ```

## Creating a Siddhi application

1. Open a text file and copy-paste following application into it.

    ```
    @App:name("SweetProduction-Store")

    @App:description('Receive events via HTTP and persist the received data in the store.')

    @Source(type = 'http', receiver.url='http://localhost:8006/productionStream', basic.auth.enabled='false',
        @map(type='json'))
    define stream insertSweetProductionStream (name string, amount double);

    @Store(type="rdbms",
           jdbc.url="jdbc:mysql://localhost:3306/production?useSSL=false",
           username="wso2si",
           password="wso2" ,
           jdbc.driver.name="com.mysql.jdbc.Driver")
    define table SweetProductionTable (name string, amount double);

    from insertSweetProductionStream
    update or insert into SweetProductionTable
    on SweetProductionTable.name == name;
    ```
    Here the `jdbc.url` parameter has the value `jdbc:mysql://localhost:3306/production?useSSL=false`. Change it to point to your MySQL server. Similarly change `username` and `password` parameters as well.

2. Save this file as `SweetProduction-Store.siddhi` in a location of your choice in the local file system.

3. Now you will execute a `CURL` command and deploy this Siddhi application. On the command line, navigate to the location where you saved the Siddhi application in above step and execute following command:
    ```
    curl -X POST "https://localhost:9443/siddhi-apps" -H "accept: application/json" -H "Content-Type: text/plain" -d @SweetProduction-Store.siddhi -u admin:admin -k
    ```

4. Upon successful deployment, you will get following response, for the `CURL` command you just executed.
    ```
    {"type":"success","message":"Siddhi App saved succesfully and will be deployed in next deployment cycle"}
    ```

5. In addition to that, you will see following log on the SI console.
    ```
    INFO {org.wso2.carbon.streaming.integrator.core.internal.StreamProcessorService} - Siddhi App SweetProduction-Store deployed successfully
    ```

    !!!info
    Next, you are going to send a few events into `insertSweetProductionStream` stream via a `CURL` command.

6. Execute following `CURL` command on the console:
    ```
    curl -X POST -d "{\"event\": {\"name\":\"Almond cookie\",\"amount\": 100.0}}"  http://localhost:8006/productionStream --header "Content-Type:application/json"
    ```

    !!!info
    You have written the Siddhi application to insert or update `SweetProductionTable` table from `insertSweetProductionStream`. As a result, above event is now inserted into the `SweetProductionTable`.

7. To verify whether above event is inserted into `SweetProductionTable`, execute following `SQL` query on the SQL console:
    ```
    SELECT * FROM SweetProductionTable;
    ```

    You can see that the event has now being inserted into the table.
    ```
    +---------------+--------+
    | name          | amount |
    +---------------+--------+
    | Almond cookie | 100.00 |
    +---------------+--------+
    ```

## Running a Siddhi Store API query

You can use 'Siddhi Store Query API' to execute queries on Siddhi Store tables.

In this tutorial, you are going to execute a simple store query via the REST API in order to fetch all records from the Siddhi Store table `SweetProductionTable`. To find out other types of queries, refer [Streaming Integrator REST API Guide](https://ei.docs.wso2.com/en/next/streaming-integrator/ref/si-rest-api-guide/).

Execute following `CURL` command on the console:
```
curl -k -X POST http://localhost:7070/stores/query -H "content-type: application/json" -u "admin:admin" -d '{"appName" : "SweetProduction-Store", "query" : "from SweetProductionTable select *" }'
```

You will get following output on the console:
```
{"records":[["Almond cookie",100.0]]}
```

## Fetching the status of a Siddhi Application

Now let's fetch the status of the Siddhi application you just deployed.

Execute following `CURL` command, on the console:
```
curl -X GET "http://localhost:9090/siddhi-apps/SweetProduction-Store/status" -H "accept: application/json" -u admin:admin -k
```

You will get following output on the command line:
```
{"status":"active"}
```

## Taking a snapshot of a Siddhi Application

In this section, we will deploy a stateful Siddhi application and use the REST API to take a snapshot of it.

1. First, enable the state persistence feature in SI server as follows. Open the `<SI_HOME>/conf/server/deployment.yaml` file on a text editor and locate the `state.persistence` section.

    ```
      # Periodic Persistence Configuration
    state.persistence:
      enabled: true
      intervalInMin: 5
      revisionsToKeep: 2
      persistenceStore: org.wso2.carbon.streaming.integrator.core.persistence.FileSystemPersistenceStore
      config:
        location: siddhi-app-persistence
    ```
    Set `enabled` parameter to `true`. Then set `intervalInMin` to `5`, and save the file.

2. Restart the Streaming Integrator server for above change to be effective.

3. Open a text file and copy-paste following application into it.
    ```
    @App:name("CountProductions")

    @App:description("Siddhi application to count the total number of orders.")

    @Source(type = 'http', receiver.url='http://localhost:8007/productionStream', basic.auth.enabled='false',
        @map(type='json'))
    define stream SweetProductionStream (name string, amount double);

    @sink(type = 'log')
    define stream LogStream (totalProductions double);

    @info(name = 'query')
    from SweetProductionStream
    select sum(amount) as totalProductions
    insert into LogStream;
    ```

4. Save this file as `CountProductions.siddhi` in a location of your choice in the local file system.

5. Now you will execute a `CURL` command and deploy this Siddhi application. On the command line, navigate to the location where you saved the Siddhi application in above step and execute following command:
    ```
    curl -X POST "https://localhost:9443/siddhi-apps" -H "accept: application/json" -H "Content-Type: text/plain" -d @CountProductions.siddhi -u admin:admin -k
    ```

6. Upon successful deployment, you will get following response, for the `CURL` command you just executed.
    ```
    {"type":"success","message":"Siddhi App saved succesfully and will be deployed in next deployment cycle"}
    ```

7. In addition to that, you will see following log on the SI console.
    ```
    INFO {org.wso2.carbon.streaming.integrator.core.internal.StreamProcessorService} - Siddhi App CountProductions deployed successfully

8. Now let's send following two sweet production events using `CURL`. Execute following two `CURL` commands on the command line:
    ```
    curl -X POST -d "{\"event\": {\"name\":\"Almond cookie\",\"amount\": 100.0}}"  http://localhost:8007/productionStream --header "Content-Type:application/json"
    ```
    ```
    curl -X POST -d "{\"event\": {\"name\":\"Baked alaska\",\"amount\": 20.0}}"  http://localhost:8007/productionStream --header "Content-Type:application/json"
    ```

9. As a result, following two lines of logs appears on the SI console:
    ```
    INFO {org.wso2.siddhi.core.stream.output.sink.LogSink} - CountProductions : LogStream : Event{timestamp=1566288572024, data=[100.0], isExpired=false}
    INFO {org.wso2.siddhi.core.stream.output.sink.LogSink} - CountProductions : LogStream : Event{timestamp=1566288596336, data=[120.0], isExpired=false}
    ```
    Notice that the current productions count is `120`.

10. Now you will invoke the Siddhi Query API to take a snapshot of the Siddhi application. Execute following `CURL` command on the command line:
    ```
    curl -X POST "https://localhost:9443/siddhi-apps/CountProductions/backup" -H "accept: application/json" -u admin:admin -k
    ```
    You will get an output similar to following:
    ```
    {"revision":"1566293390654__CountProductions"}
    ```
    !!! info
        `1566293390654__CountProductions` is the revision number of the Siddhi application snapshot that you requested through the REST API. You can store this revision number and later use it in order to restore the Siddhi application to the state at which you took the snapshot.

## Restoring a Siddhi Application via a snapshot

In the previous section, you took a snapshot of the `CountProductions` Siddhi application when the productions count was `120`. Now you are going to increase the count further by sending a few more production events and then restore the Siddhi application to the state you backed up.

1. Send following two sweet production events:
    ```
    curl -X POST -d "{\"event\": {\"name\":\"Cup cake\",\"amount\": 300.0}}"  http://localhost:8007/productionStream --header "Content-Type:application/json"
    ```
    ```
    curl -X POST -d "{\"event\": {\"name\":\"Doughnut\",\"amount\": 500.0}}"  http://localhost:8007/productionStream --header "Content-Type:application/json"
    ```

2. As a result, following two lines of logs appears on the SI console:
    ```
    INFO {org.wso2.siddhi.core.stream.output.sink.LogSink} - CountProductions : LogStream : Event{timestamp=1566288572024, data=[420.0], isExpired=false}
    INFO {org.wso2.siddhi.core.stream.output.sink.LogSink} - CountProductions : LogStream : Event{timestamp=1566288596336, data=[920.0], isExpired=false}
    ```
    Notice that the current productions count is `920`.

3. Now you will invoke the Siddhi Query API to restore the snapshot that you obtained in step 10 of [Taking a snapshot of a Siddhi Application](#Taking-a-snapshot-of-a-Siddhi-Application) section of this tutorial.

    In this example, the revision number obtained was `1566293390654__CountProductions` (refer step 10 in [Taking a snapshot of a Siddhi Application](#Taking-a-snapshot-of-a-Siddhi-Application) section.). When restoring the state, use the exact revision number which you obtained.

    Execute following command on the command line:
    ```
    curl -X POST "https://localhost:9443/siddhi-apps/CountProductions/restore?revision=1566293390654__CountProductions" -H "accept: application/json" -u admin:admin -k
    ```
    !!! note
        Replace `1566293390654__CountProductions` with the revision number which you obtained when taking the Siddhi application snapshot.

    You will receive following response:
    ```
    {"type":"success","message":"State restored to revision 1566293390654__CountProductions for Siddhi App :CountProductions"}
    ```
    In addition to that, following log will be printed on the SI console:
    ```
    INFO {org.wso2.carbon.streaming.integrator.core.persistence.FileSystemPersistenceStore} - State loaded for CountProductions revision 1566293390654__CountProductions from the file system.
    ```

4.  Now send another sweet production event by executing following `CURL` command:
    ```
    curl -X POST -d "{\"event\": {\"name\":\"Danish pastry\",\"amount\": 100.0}}"  http://localhost:8007/productionStream --header "Content-Type:application/json"
    ```

5. As a result, following log appears on the SI console:
    ```
    INFO {org.wso2.siddhi.core.stream.output.sink.LogSink} - CountProductions : LogStream : Event{timestamp=1566293520176, data=[220.0], isExpired=false}
    ```
    Notice that the productions count is `220`. This is because the count got set to `120` when you restored the snapshot.
