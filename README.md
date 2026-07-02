# docker_setup


# Local Database Infrastructure: MySQL Setup via Docker Compose & DBeaver

Welcome to this step-by-step setup guide! This repository walks through how to configure, automate, and connect an isolated **MySQL Database** inside a **Docker Container**, using **VS Code** as our code workspace and **DBeaver** as our visual database control panel.

---

## 🗺️ System Architecture: The Big Picture

When developing software, installing databases directly onto your laptop creates system clutter, massive file footprints, and version conflicts. Instead, we use a modern **decoupled architecture**:

1. **The Infrastructure Layer (Docker):** The actual database motor runs completely isolated inside a lightweight virtual environment box.
2. **The Code Layer (VS Code):** Our code editor where we store configuration files like `docker-compose.yml`.
3. **The Presentation Layer (DBeaver):** An external database Graphic User Interface (GUI) client. DBeaver acts as a dashboard wrapper so we can see tables, run queries, and visualize data without using a black command-line screen.

```text
+------------------------------------------+               +----------------------------------------+
|               YOUR LAPTOP                |               |            DOCKER CONTAINER            |
|                                          |               |                                        |
|  [ DBeaver GUI Client ]                  |               |  [ MySQL Server Instance ]             |
|   Visualizes tables & runs queries       |               |   Runs completely isolated inside box  |
|   Target Host Connection: localhost      |               |   Internal Port: 3306                  |
|   Target Port Mapping:    3306           | -- Network -> |                                        |
+------------------------------------------+               +----------------------------------------+
```

⚡ Essential Docker Commands Cheatsheet
Here are the only commands you need to know to manage this entire setup from your terminal:

```text
# 1. Check which base templates (images) are already downloaded on your laptop
docker images

# 2. Start your entire database environment in the background
docker compose up -d

# 3. Monitor live database logs (Crucial to see when MySQL is fully booted and ready)
docker compose logs db

# 4. View actively running containers and see their port mappings
docker ps

# 5. Stop running containers and cleanly disconnect the virtual network switches
docker compose down

# 6. Completely wipe out containers AND their saved data storage for a total reset
docker compose down -v
```
📄 The Configuration Blueprint (docker-compose.yml)
Instead of typing long, messy terminal strings every time you want to start a database, we use Infrastructure as Code (IaC).

Create a file named exactly docker-compose.yml in your project folder using VS Code, and paste the following content inside it:
```text
YAML
services:
  # --- Service: Relational Database ---
  db:
    image: mysql:8.0.36
    container_name: mysql_test_container
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: root_password_changeme
      MYSQL_DATABASE: test
      MYSQL_USER: pooja
      MYSQL_PASSWORD: password
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql
    networks:
      - dev_network

volumes:
  mysql_data:
    name: mysql_test_data

networks:
  dev_network:
    name: local_dev_network
```
🔍 Parameter Explanations for Beginners:

image: mysql:8.0.36: Tells Docker the exact version blueprint to download. Picking a specific version number guarantees the database works identically on everyone's computer.

environment:: Variables passed into the container box. The MySQL template reads these to automatically create our database name (test), our user account (pooja), and our passwords.

ports: - "3306:3306": The Network Bridge. MySQL naturally listens on port 3306 inside its hidden container. This line bridges it to port 3306 on your actual laptop so tools like DBeaver can access it.

volumes: - mysql_data:/var/lib/mysql: The Data Anchor. Containers are temporary and wipe their data on shutdown. By linking a named volume (mysql_data) directly to MySQL's internal storage folder (/var/lib/mysql), your data remains permanently saved on your physical hard drive.

🔌 Step-by-Step Connection Setup (DBeaver)
Once your container is running (docker compose up -d), wait 15 seconds for MySQL to initialize inside Docker Desktop. Then, configure your DBeaver tool:

Open DBeaver and click New Database Connection -> Select MySQL.

Fill in the connection settings fields exactly as follows:

Host: localhost (or 127.0.0.1)

Port: 3306

Database: test

Username: username

Password: password

Complete Connection URL String
If you prefer pasting a single connection string directly into DBeaver's URL input block, use this:

jdbc:mysql://localhost:3306/test?user=pooja&password=password&allowPublicKeyRetrieval=true&useSSL=false

⚠️ Crucial MySQL 8+ Fix: MySQL 8 uses an advanced password encryption protocol. When connecting external visual tools, you must go to the Driver Properties tab in DBeaver, find allowPublicKeyRetrieval and set it to TRUE, and set useSSL to FALSE. Otherwise, your connection will drop immediately!

📊 Initializing the Data Model Schema
Once connected via DBeaver, open an empty SQL Editor tab (Ctrl + Alt + T) and run this script to create a secure, data-validated student table:

```text
SQL
CREATE TABLE student (
    student_id INT AUTO_INCREMENT PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    email VARCHAR(100) NOT NULL UNIQUE,
    age INT,
    enrollment_date DATE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    -- Safety check barriers to protect data integrity at the database layer
    CONSTRAINT chk_student_age CHECK (age >= 0 AND age <= 120),
    CONSTRAINT chk_student_email CHECK (email LIKE '%_@__%.__%')
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
Advanced Schema Highlights:
utf8mb4: Full native support for international text characters, accents, and emojis.
```

ON UPDATE CURRENT_TIMESTAMP: Native database-level auditing. The database automatically logs exactly when a row changes without requiring custom code in a backend application layer.

CONSTRAINT Blocks: Infrastructure defense lines that instantly reject invalid entries (like a negative age value or a broken email syntax pattern) before they corrupt your tables.

🚀 Scaling Beyond MySQL: Adding More Services
One of the greatest benefits of using Docker Compose is scalability. If your local development requirements expand tomorrow—for instance, if you need to integrate a real-time event streaming platform like Apache Kafka—you do not need separate infrastructure files.

You simply add the new services directly underneath your existing MySQL service inside the same services: block of your docker-compose.yml file:

YAML
services:
  # MySQL Service remains here...
  db:
    image: mysql:8.0.36
    # ... configurations ...

  # New Service: Kafka Coordinator
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    container_name: zookeeper_container
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
    networks:
      - dev_network

  # New Service: Kafka Event Streaming Broker
  kafka:
    image: confluentinc/cp-kafka:7.5.0
    container_name: kafka_container
    ports:
      - "9092:9092"
    environment:
      KAFKA_ZOOKEEPER_CONNECT: 'zookeeper:2181'
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092,PLAINTEXT_INTERNAL://kafka:29092
    networks:
      - dev_network
    depends_on:
      - zookeeper
By adopting this structure, running docker compose up -d handles everything simultaneously. Docker connects all services onto the exact same local virtual network automatically, matching standard enterprise microservice patterns.

🧠 Enterprise Context: How Real Companies Handle Data
In corporate environments, live customer data is never downloaded onto a local developer machine due to privacy compliance laws (like GDPR or HIPAA) and massive multi-terabyte data sizes.

Instead, corporate engineering pipelines mimic this exact Docker workflow using:

Mock Data Seeds: Structured .sql scripts that drop clean, realistic, dummy datasets into local Docker volumes for feature testing.

Data Masking: Automated scripts that take database snapshots, scramble real personal names/credit cards, and output safe, compliant local testing files.

Cloud Development Database Clusters: Running application environments locally inside Docker containers while using environment parameters to securely route application connections to a shared cloud database testing cluster (AWS/Azure) over a secure company VPN.
