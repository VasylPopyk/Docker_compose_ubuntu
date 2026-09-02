# Isolated WordPress & MySQL Deployment using Docker Compose

A multi-container web application stack designed as an Infrastructure as Code (IaC) deployment. This project demonstrates declarative container orchestration, host-level network isolation, persistent data storage, and operational logging workflows using Docker Compose on Kali Linux.

---

## 🛠️ Architecture & Core Components

The infrastructure consists of two decoupled services running inside an isolated internal bridge network:

                  +-----------------------------------+
                  |            Host Machine           |
                  |         (Port 8080 Exposed)       |
                  +-----------------+-----------------+
                                    |
                                    v
+-----------------------------------:-----------------------------------+
| Bridge Network: wp-network        | (Port 80)                         |
|                                   v                                   |
|  +--------------------------------+--------------------------------+  |
|  | Container: wp_frontend (WordPress:latest)                     |  |
|  | - Volume: wp_data -> /var/www/html/wp-content                 |  |
|  +--------------------------------+--------------------------------+  |
|                                   |                                   |
|                                   | Internal DNS (`db:3306`)          |
|                                   v                                   |
|  +--------------------------------+--------------------------------+  |
|  | Container: wp_database (MySQL:5.7)                            |  |
|  | - Volume: db_data -> /var/lib/mysql                             |  |
|  +-----------------------------------------------------------------+  |
|                                                                       |
+-----------------------------------------------------------------------+

### Key Engineering Decisions

* **Network Isolation (Least Privilege):** The MySQL database container (`wp_database`) is strictly confined to `wp-network` and exposes no ports to the host interface. The WordPress frontend (`wp_frontend`) communicates with MySQL via Docker's internal DNS resolver on port `3306`.
* **State Persistence:** Persistent storage is guaranteed through named volumes (`db_data` and `wp_data`). All database schemas, user state, and uploaded media files persist across container teardowns (`docker compose down`).
* **Declarative Infrastructure:** The entire stack lifecycle, network definitions, environment credentials, and mount targets are managed via `docker-compose.yml`.

---

## 📄 Infrastructure Configuration (`docker-compose.yml`)

```yaml
version: '3.8'

services:
  db:
    image: mysql:5.7
    container_name: wp_database
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: SecureRootPassword123!
      MYSQL_DATABASE: wordpress_db
      MYSQL_USER: wp_user
      MYSQL_PASSWORD: WPUserPassword123!
    volumes:
      - db_data:/var/lib/mysql
    networks:
      - wp-network

  wordpress:
    image: wordpress:latest
    container_name: wp_frontend
    restart: always
    ports:
      - "8080:80"
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: wp_user
      WORDPRESS_DB_PASSWORD: WPUserPassword123!
      WORDPRESS_DB_NAME: wordpress_db
    volumes:
      - wp_data:/var/www/html/wp-content
    depends_on:
      - db
    networks:
      - wp-network

volumes:
  db_data:
  wp_data:

networks:
  wp-network:
    driver: bridge
