FROM postgres:17.2-alpine

COPY 01_CreateScheme.sql /docker-entrypoint-initdb.d/
COPY 02_InsertData.sql /docker-entrypoint-initdb.d/