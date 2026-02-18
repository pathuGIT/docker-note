
# Full-Stack Project Docker Guide

This guide explains how to run your **full-stack project** in **development** and **production**, both with **local MySQL** and **Azure MySQL Flexible Server**.

---

## **1️⃣ Development Setup**

During development, you can run Docker containers to **test and develop** your app with **hot reload**.

### **Project Structure**

```
full-stack-project/
│
├─ client/                     # React app
│   ├─ src/
│   ├─ public/
│   ├─ package.json
│   ├─ Dockerfile
│   ├─ .dockerignore
│   └─ vite.config.js
│
├─ server/                     # Node.js server
│   ├─ controller/
│   ├─ service/
│   ├─ routes/
│   ├─ package.json
│   ├─ Dockerfile
│   ├─ .dockerignore
│   └─ server.js
│
├─ docker-compose.dev.yml      # Development Docker Compose
├─ init-db/
│   └─ init.sql                # Optional database initialization
└─ README.md
```

---

### **Client Dockerfile (`client/Dockerfile`)**

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 5173
CMD ["npm", "run", "dev", "--", "--host"]
```

### **Client `.dockerignore`**

```
node_modules/
package-lock.json
.env
```

---

### **Server Dockerfile (`server/Dockerfile`)**

```dockerfile
FROM node:20-alpine
RUN npm install -g nodemon
WORKDIR /app
COPY . .
RUN npm install
EXPOSE 5000
CMD ["npm", "start"]
```

### **Server `.dockerignore`**

```
node_modules/
package-lock.json
.env
```

---

### **Development Docker Compose with Local MySQL**

```yaml
version: '3.8'

services:
  # MySQL Database
  mysql:
    image: mysql:8.0
    container_name: mysql-db
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: usersdb
      MYSQL_USER: appuser
      MYSQL_PASSWORD: apppassword
    ports:
      - "3307:3306"
    volumes:
      - mysql-data:/var/lib/mysql
      - ./init-db:/docker-entrypoint-initdb.d
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "127.0.0.1", "-u", "root", "-prootpassword"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Node.js Backend Server
  server:
    build: ./server
    container_name: node-server
    restart: always
    ports:
      - "5000:5000"
    environment:
      DB_HOST: mysql
      DB_USER: appuser
      DB_PASSWORD: apppassword
      DB_NAME: usersdb
      DB_PORT: 3306
    depends_on:
      mysql:
        condition: service_healthy
    volumes:
      - ./server:/app
      - /app/node_modules
    networks:
      - app-network

  # React Frontend Client
  client:
    build: ./client
    container_name: react-client
    restart: always
    ports:
      - "5173:5173"
    environment:
      - CHOKIDAR_USEPOLLING=true
      - VITE_API_URL=http://localhost:5000
    depends_on:
      - server
    volumes:
      - ./client:/app
      - /app/node_modules
    stdin_open: true
    tty: true
    networks:
      - app-network

networks:
  app-network:
    driver: bridge

volumes:
  mysql-data:
```

---

### **Development Docker Compose with Azure MySQL**

If you are using **Azure MySQL Flexible Server**, remove the local MySQL service and connect server directly to Azure:

```yaml
version: '3.8'

services:
  server:
    build: ./server
    container_name: node-server
    restart: always
    ports:
      - "5000:5000"
    environment:
      DB_HOST: <AZURE_MYSQL_HOST>        # e.g., myserver.mysql.database.azure.com
      DB_USER: <AZURE_DB_USER>           # e.g., appuser@myserver
      DB_PASSWORD: <AZURE_DB_PASSWORD>
      DB_NAME: usersdb
      DB_PORT: 3306
      NODE_ENV: development
    volumes:
      - ./server:/app
      - /app/node_modules
    networks:
      - app-network

  client:
    build: ./client
    container_name: react-client
    restart: always
    ports:
      - "5173:5173"
    environment:
      - CHOKIDAR_USEPOLLING=true
      - VITE_API_URL=http://localhost:5000
    depends_on:
      - server
    volumes:
      - ./client:/app
      - /app/node_modules
    stdin_open: true
    tty: true
    networks:
      - app-network

networks:
  app-network:
    driver: bridge
```

---

### **Commands for Development**

```bash
# Build Docker images
docker compose -f docker-compose.dev.yml build

# Run containers in detached mode
docker compose -f docker-compose.dev.yml up -d
```

---


## **2️⃣ Production Setup**

Ah! I get it now — you want the **minimal file structure for production deployment on a VM** (or another machine) **before actually running the containers**, basically what you’d copy or clone to the VM.

Here’s a clear layout:

```
docker-full-test/
│
├─ docker-compose.prod.yml      # Production Docker Compose file
└─ init-db/                     # Optional database initialization scripts (only if using local MySQL)
    └─ init.sql
```

### **Notes**

1. **If using Azure MySQL Flexible Server**, the `init-db` folder is **not needed** — the database already exists in Azure.

--- 
After building and pushing images, use this **production configuration** for deployment.

### **Production Docker Compose with Local MySQL**

```yaml
version: "3.8"

services:
  mysql:
    image: mysql:8.0
    container_name: mysql-db
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: usersdb
      MYSQL_USER: appuser
      MYSQL_PASSWORD: apppassword
    ports:
      - "3307:3306"
    volumes:
      - mysql-data:/var/lib/mysql
    networks:
      - app-network

  server:
    image: lakshandocr/docer-compose-server:latest
    container_name: node-server
    restart: always
    ports:
      - "5000:5000"
    environment:
      DB_HOST: mysql
      DB_USER: appuser
      DB_PASSWORD: apppassword
      DB_NAME: usersdb
      DB_PORT: 3306
    depends_on:
      - mysql
    networks:
      - app-network

  client:
    image: lakshandocr/docer-compose-client:latest
    container_name: react-client
    restart: always
    ports:
      - "5173:5173"
    environment:
      - VITE_API_URL=http://localhost:5000
    depends_on:
      - server
    networks:
      - app-network

networks:
  app-network:
    driver: bridge

volumes:
  mysql-data:
```

---

### **Production Docker Compose with Azure MySQL**

```yaml
version: "3.8"

services:
  server:
    image: lakshandocr/docer-compose-server:latest
    container_name: node-server
    restart: always
    ports:
      - "5000:5000"
    environment:
      DB_HOST: <AZURE_MYSQL_HOST>
      DB_USER: <AZURE_DB_USER>
      DB_PASSWORD: <AZURE_DB_PASSWORD>
      DB_NAME: usersdb
      DB_PORT: 3306
      NODE_ENV: production
    networks:
      - app-network

  client:
    image: lakshandocr/docer-compose-client:latest
    container_name: react-client
    restart: always
    ports:
      - "5173:5173"
    environment:
      VITE_API_URL: http://<AZURE_VM_PUBLIC_IP>:5000
    depends_on:
      - server
    networks:
      - app-network

networks:
  app-network:
    driver: bridge
```

---

### **Commands for Production**

```bash
# Push images
docker compose -f docker-compose.prod.yml push

# Remove all existing containers and volumes (optional)
docker compose -f docker-compose.prod.yml down -v

# Pull latest images
docker compose -f docker-compose.prod.yml pull

# Start all containers in detached mode
docker compose -f docker-compose.prod.yml up -d
```

