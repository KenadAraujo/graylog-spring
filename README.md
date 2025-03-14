# Projeto teste do Graylog e do SpringBoot
<img src="https://img.shields.io/badge/Java-ED8B00?style=for-the-badge&logo=openjdk&logoColor=white" /> <img src="https://img.shields.io/badge/docker-%230db7ed.svg?style=for-the-badge&logo=docker&logoColor=white" />

É um projeto teste para Dockerizar uma aplicação SpringBoot enviando logs para o Graylog. Nessa aplicação, foram criadas duas stacks: a primeira representa a stack do Graylog e a segunda a stack do App em [Spring Boot](https://github.com/KenadAraujo/docker-teste).

## Dockerfile do App

Abaixo segue um exemplo do Dockerfile para o app que será utilizado posteriormente na stack do app:

```Docker
FROM openjdk:17
WORKDIR /app
COPY . /app
ARG PORT=9000
ENV PORT=$PORT
EXPOSE $PORT
RUN chmod +x ./mvnw
RUN ./mvnw clean package
CMD java -jar target/*.jar
```
Comando para compilar a imagem


```bash
docker run -it -p 9000:9000 kenadaraujo/teste:1.0
```

## docker-compose do graylog

Esse é um docker-compose básico para o Graylog, note que eu desativei as politicas de seguranças para fim de teste, caso não desative, ele irá exigir uma senha forte e um certificado para https:

```yaml
version: '3.8'

services:
  mongo:
    image: mongo:6.0
    container_name: graylog-mongo
    restart: unless-stopped
    environment:
      - MONGO_INITDB_ROOT_USERNAME=admin
      - MONGO_INITDB_ROOT_PASSWORD=adminpass
    networks:
      - graylog-net
    healthcheck:
      test: echo 'db.runCommand("ping").ok' | mongosh --quiet
      interval: 10s
      retries: 5
      start_period: 30s

  opensearch:
    image: opensearchproject/opensearch:2.19.1
    container_name: graylog-opensearch
    restart: unless-stopped
    environment:
      - discovery.type=single-node
      - OPENSEARCH_JAVA_OPTS=-Xms1g -Xmx1g
      - bootstrap.memory_lock=true
      - OPENSEARCH_INITIAL_ADMIN_PASSWORD=4iEUS!Y6N7
      - DISABLE_SECURITY_PLUGIN=true
    ulimits:
      memlock:
        soft: -1
        hard: -1
    networks:
      - graylog-net
    ports:
      - 9200:9200 # REST API
      - 9600:9600 # Performance Analyzer
    healthcheck:
      test: ["CMD-SHELL", "curl -k -u admin:4iEUS!Y6N7 --silent --fail http://graylog-opensearch:9200/_cluster/health || exit 1"]
      interval: 20s
      retries: 15
      start_period: 60s

  graylog:
    image: graylog/graylog:6.1.8
    container_name: graylog
    restart: unless-stopped
    depends_on:
      mongo:
        condition: service_healthy
      opensearch:
        condition: service_healthy
    environment:
      - GRAYLOG_PASSWORD_SECRET=8c6976e5b5410415bde908bd4dee15dfb167a9c873fc4bb8a81f6f2ab448a918
      - GRAYLOG_ROOT_PASSWORD_SHA2=8c6976e5b5410415bde908bd4dee15dfb167a9c873fc4bb8a81f6f2ab448a918
      - GRAYLOG_HTTP_BIND_ADDRESS=0.0.0.0:9000
      - GRAYLOG_HTTP_EXTERNAL_URI=http://localhost:9000/
      - GRAYLOG_MONGODB_URI=mongodb://admin:adminpass@graylog-mongo:27017/graylog?authSource=admin
      - GRAYLOG_ELASTICSEARCH_HOSTS=http://graylog-opensearch:9200
      - GRAYLOG_ELASTICSEARCH_USERNAME=admin
      - GRAYLOG_ELASTICSEARCH_PASSWORD=4iEUS!Y6N7
    networks:
      - graylog-net
    ports:
      - "9000:9000"   # Interface Web
      - "1514:1514"   # Entrada Syslog TCP
      - "1514:1514/udp" # Entrada Syslog UDP
      - "12201:12201" # Entrada GELF TCP
      - "12201:12201/udp" # Entrada GELF UDP
    
networks:
  graylog-net:
    driver: bridge
```

Comando para subir a stack

```bash
docker-compose up
```

Comando para destruir a stack
```bash
docker-compose down
```

### docker-compose do app

Esse é um docker-compose para enviar os logs via DRIVER GELF:

```yaml
version: '3.8'

services:
  app:
    image: kenadaraujo/teste:1.0
    container_name: graylog_app
    restart: unless-stopped
    networks:
      - app-net
    ports:
      - 5000:5000
    logging:
      driver: gelf
      options:
        gelf-address: "udp://localhost:12201"
        tag: "graylog_app"

networks:
  app-net:
    driver: bridge
```

Para executar utilize o comando abaixo

```bash
docker compose -f docker-compose-app.yml up
```