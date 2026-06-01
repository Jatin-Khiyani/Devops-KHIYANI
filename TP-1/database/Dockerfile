FROM postgres:17.2-alpine

ENV POSTGRES_DB=db \
   POSTGRES_USER=usr \
   POSTGRES_PASSWORD=pwd

COPY 01_CreateScheme.sql /docker-entrypoint-initdb.d/
COPY 02_InsertData.sql /docker-entrypoint-initdb.d/