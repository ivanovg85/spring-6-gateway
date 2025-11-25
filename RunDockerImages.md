# Running Multi-Container Spring Boot Application

This guide provides Docker commands to run the complete multi-container Spring Boot setup with OAuth2 authentication, API Gateway, and multiple microservices.

## Architecture Overview

The setup consists of:
- **Auth Server** - OAuth2 authentication server (port 9000)
- **Gateway** - API Gateway entry point (port 8080)
- **REST MVC** - Traditional REST API with MySQL database
- **Reactive** - Reactive WebFlux API with H2 database
- **Reactive Mongo** - Reactive API with MongoDB database
- **MySQL** - Database for REST MVC service
- **MongoDB** - Database for Reactive Mongo service

## Docker Commands to Run Multi-Container Setup

### 1. Create a Docker Network
```bash
docker network create spring-network
```

### 2. Start MySQL Database (for rest-mvc)
```bash
docker run -d \
  --name mysql \
  --network spring-network \
  -e MYSQL_ROOT_PASSWORD=root \
  -e MYSQL_DATABASE=restdb \
  -e MYSQL_USER=restadmin \
  -e MYSQL_PASSWORD=password \
  -p 3306:3306 \
  mysql:8.0
```

### 3. Start MongoDB Database (for reactive-mongo)
```bash
docker run -d \
  --name mongodb \
  --network spring-network \
  -e MONGO_INITDB_ROOT_USERNAME=root \
  -e MONGO_INITDB_ROOT_PASSWORD=example \
  -p 27017:27017 \
  mongo:latest
```

### 4. Start Auth Server
```bash
docker run -d \
  --name auth-server \
  --network spring-network \
  -p 9000:9000 \
  spring-6-auth-server:0.0.1-SNAPSHOT
```

### 5. Start REST MVC Service
```bash
docker run -d \
  --name rest-mvc \
  --network spring-network \
  -p 8081:8080 \
  -e SPRING_PROFILES_ACTIVE=localmysql \
  -e "SPRING_DATASOURCE_URL=jdbc:mysql://mysql:3306/restdb?useUnicode=true&characterEncoding=UTF-8&serverTimezone=UTC" \
  -e "SPRING_SECURITY_OAUTH2_RESOURCESERVER_JWT_ISSUER_URI=http://auth-server:9000" \
  -e SERVER_PORT=8080 \
  spring-6-rest-mvc:0.0.1-SNAPSHOT
```

### 6. Start Reactive Service
```bash
docker run -d \
  --name reactive \
  --network spring-network \
  -p 8082:8080 \
  -e "SPRING_SECURITY_OAUTH2_RESOURCESERVER_JWT_ISSUER_URI=http://auth-server:9000" \
  -e SERVER_PORT=8080 \
  spring-6-reactive:0.0.1-SNAPSHOT
```

### 7. Start Reactive Mongo Service
```bash
docker run -d \
  --name reactive-mongo \
  --network spring-network \
  -p 8083:8080 \
  -e SPRING_DATA_MONGODB_HOST=mongodb \
  -e "SPRING_SECURITY_OAUTH2_RESOURCESERVER_JWT_ISSUER_URI=http://auth-server:9000" \
  -e SERVER_PORT=8080 \
  spring-6-reactive-mongo:0.0.1-SNAPSHOT
```

### 8. Start Gateway (Entry Point)
```bash
docker run -d \
  --name gateway \
  --network spring-network \
  -p 8080:8080 \
  -e SPRING_PROFILES_ACTIVE=docker \
  spring-6-gateway:0.0.1-SNAPSHOT
```

## Access Points

Once all containers are running:
- **Gateway (Main Entry Point)**: http://localhost:8080
  - REST MVC API: http://localhost:8080/api/v1/**
  - Reactive API: http://localhost:8080/api/v2/**
  - Reactive Mongo API: http://localhost:8080/api/v3/**
  - OAuth2 endpoints: http://localhost:8080/oauth2/**
- **Auth Server**: http://localhost:9000
- **Direct Service Access** (for debugging):
  - REST MVC: http://localhost:8081
  - Reactive: http://localhost:8082
  - Reactive Mongo: http://localhost:8083

## Useful Management Commands

### View Running Containers
```bash
docker ps
```

### View Logs
```bash
# View logs for a specific service
docker logs -f gateway
docker logs -f auth-server
docker logs -f rest-mvc
docker logs -f reactive
docker logs -f reactive-mongo
docker logs -f mysql
docker logs -f mongodb
```

### Stop All Containers
```bash
docker stop gateway rest-mvc reactive reactive-mongo auth-server mongodb mysql
```

### Remove All Containers
```bash
docker rm gateway rest-mvc reactive reactive-mongo auth-server mongodb mysql
```

### Remove Network
```bash
docker network rm spring-network
```

### Full Cleanup and Restart
```bash
# Stop all containers
docker stop gateway rest-mvc reactive reactive-mongo auth-server mongodb mysql

# Remove all containers
docker rm gateway rest-mvc reactive reactive-mongo auth-server mongodb mysql

# Remove network
docker network rm spring-network

# Then run all the setup commands again from step 1
```

## Important Notes

1. **Startup Order**: The services are started in dependency order:
   - Databases first (MySQL, MongoDB)
   - Auth server second
   - Application services third
   - Gateway last

2. **Initialization Time**: Wait 10-20 seconds after starting all containers for services to fully initialize before accessing them through the gateway.

3. **Gateway Profile**: The gateway uses the `docker` profile (application-docker.yml) which routes to services using container names instead of localhost.

4. **Database Migrations**: The REST MVC service uses Flyway for database migrations, which will run automatically on startup.

5. **Health Checks**: All services expose health check endpoints at `/actuator/health` for monitoring.

## Troubleshooting

### Service Not Responding
```bash
# Check if container is running
docker ps

# Check container logs
docker logs <container-name>

# Restart a specific service
docker restart <container-name>
```

### Network Issues
```bash
# Inspect the network
docker network inspect spring-network

# Verify all containers are on the same network
docker network inspect spring-network | grep Name
```

### Database Connection Issues
```bash
# For MySQL
docker exec -it mysql mysql -u restadmin -p

# For MongoDB
docker exec -it mongodb mongosh -u root -p
```
