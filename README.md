# Restaurant data migration with Spring Batch

A small Spring Batch job that copies restaurant documents from MongoDB into a relational Postgres table, flattening and summarising them on the way.

It is a learning project. It demonstrates one thing: the reader, processor and writer split of a chunk-oriented Spring Batch step, moving data between two different kinds of database. It has no REST API, no scheduling and no production hardening.

## Quick start

Prerequisites: Java 17 or newer (the pom targets 17; I ran it on 21), Docker. The Maven wrapper is included.

1. Start MongoDB and Postgres with the credentials that `src/main/resources/application.yaml` expects (local development values only):

   ```bash
   docker run -d --name mongodb -p 27017:27017 \
     -e MONGO_INITDB_ROOT_USERNAME=admin -e MONGO_INITDB_ROOT_PASSWORD=password mongo

   docker run -d --name postgres -p 5432:5432 \
     -e POSTGRES_USER=admin -e POSTGRES_PASSWORD=password -e POSTGRES_DB=sample_restaurants postgres
   ```

2. Load the sample dataset (3,772 restaurants in `restaurants.json`) into the `restaurants` collection of the `sample_restaurants` database:

   ```bash
   docker cp restaurants.json mongodb:/tmp/restaurants.json
   docker exec mongodb mongoimport --username admin --password password --authenticationDatabase admin \
     --db sample_restaurants --collection restaurants --file /tmp/restaurants.json
   ```

3. Clone the project and build it:

   ```bash
   git clone git@github.com:sharanggupta/batch-data-migration.git
   cd batch-data-migration
   ./mvnw clean install
   ```

   Both databases must be running and loaded first. The build runs the tests, and the one test starts the full application, which launches the job (see below). So this step already performs the migration.

4. Check the result:

   ```bash
   docker exec postgres psql -U admin -d sample_restaurants -c "select count(*) from destination_restaurant_entity"
   ```

   Expect 3772.

## Data flow

The job `importUserJob` has one step, `step1`, that processes 10 documents per chunk. Spring Boot launches it on startup.

| Part | Class | What it does |
|---|---|---|
| Reader | `SourceRestaurantReader` | A `MongoPagingItemReader` that reads every document in the `restaurants` collection into `SourceRestaurantEntity`. |
| Processor | `MongoToPostgresProcessor` | Flattens the nested address into `building`, `street` and `zipcode`, computes `averageScore` from the list of grades (0.0 when there are none), and keeps the date of the latest grade. |
| Writer | `DestinationRestaurantWriter` | A `JpaItemWriter` that inserts `DestinationRestaurantEntity` rows into Postgres. |
| Job and step | `BatchConfig` | Wires the three together and logs when the step starts and finishes. |

Spring Batch keeps its own bookkeeping tables (`batch_job_instance`, `batch_step_execution` and so on) in Postgres next to the data. Hibernate creates the `destination_restaurant_entity` table (`ddl-auto: update`).

A spot check: the restaurant with `restaurant_id` 30075445 (Morris Park Bake Shop) has grade scores 2, 6, 10, 9 and 14, and lands in Postgres with `average_score` 8.2 and `last_grade_date` 2014-03-03.

## Running it again

`./mvnw spring-boot:run` starts the application and launches the job, but with no job parameters it is the same job instance that already completed, so Spring Batch logs "Step already complete" and inserts nothing. To migrate from scratch, remove the Postgres container (the batch tables live in it) and repeat from step 1. 

## Tests

There is one test, `BatchDataMigrationApplicationTests.contextLoads`. It needs both databases because it starts the whole application. There are no unit tests for the processor, and there is no CI workflow.

## Limitations

- Local credentials are hard-coded in `application.yaml`; do not reuse them anywhere real.
- Reading the processor, a document without an `address` would throw a `NullPointerException`; the sample data never hits this.
- The job takes no parameters, so each database can be migrated once. There is no incremental loading.
