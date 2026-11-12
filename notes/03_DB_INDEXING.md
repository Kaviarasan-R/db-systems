# DB INDEXING

## Setup

```shell
docker pull postgres:latest
docker run -d --name indexing -e POSTGRES_PASSWORD=postgres postgres:latest
docker exec -it indexing psql -U postgres
```

Populate 1 million rows

```psql
create table temp(t int);
insert into temp(t) select random() * 100 from generate_series(0, 1000000);
```

## Indexing