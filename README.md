
# Database Migration

Data migration project from NoSQL to SQL using spring batch

## Setup

### source mongodb database
```bash
docker pull mongo:latest
docker run -d -p 27017:27017 --name mongodb -e MONGO_INITDB_ROOT_USERNAME=admin -e MONGO_INITDB_ROOT_PASSWORD=password mongo

```
### destination postgre database
```bash
docker pull postgres
docker run --name postgres -e POSTGRES_USER=admin -e POSTGRES_PASSWORD=password -e POSTGRES_DB=sample_restaurants -p 5432:5432 -d postgres

```    
Create a new database and collection (sample_restaurants) and import data (restaurants.json) into mongodb using mongoimport or MongoDB Compass

## Run Locally

Clone the project

```bash
git clone git@github.com:sharanggupta/batch-data-migration.git
```

Go to the project directory

```bash
cd batch-data-migration
```

Install dependencies

```bash
./mvnw clean install
```

Start the migration

```bash
./mvnw spring-boot:run 
```

