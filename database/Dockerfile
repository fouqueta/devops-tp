FROM postgres:14.1-alpine

ENV POSTGRES_DB=db \
   POSTGRES_USER=usr

COPY ./sql-scripts /docker-entrypoint-initdb.d
