# 🚀 Metabase Automatic Docker Integration Script

<p align="center">
  <img src="https://img.shields.io/badge/Metabase-509EE3.svg?style=for-the-badge&logo=Metabase&logoColor=white"/>
  <img src="https://img.shields.io/badge/PostgreSQL-4169E1.svg?style=for-the-badge&logo=PostgreSQL&logoColor=white"/>
  <img src="https://img.shields.io/badge/shell_script-%23121011.svg?style=for-the-badge&logo=gnu-bash&logoColor=white"/>
  <img src="https://img.shields.io/badge/Linux-FCC624.svg?style=for-the-badge&logo=Linux&logoColor=black"/>
  <img src="https://img.shields.io/badge/Docker-2496ED.svg?style=for-the-badge&logo=Docker&logoColor=white"/>
  <img src="https://img.shields.io/badge/GNU%20Bash-4EAA25.svg?style=for-the-badge&logo=GNU-Bash&logoColor=white"/>
</p>


### Simplify Your Metabase Setup with a Single Script 🛠️

Welcome to the **Metabase Automatic Docker Integration Script** repository! This handy Bash script automates the process of integrating PostgreSQL databases into your Metabase Docker container. By interacting with the Metabase API, this script streamlines database management, saving you time and reducing manual configuration.

---

## ✨ Features

This script does the heavy lifting for you:
1. **Automatic Network Configuration**: Fetches the gateway IP of your specified Docker network.
2. **Seamless Authentication**: Authenticates with the Metabase API to get a session token.
3. **Database Integration**: Adds your PostgreSQL databases to Metabase with just one command.

## 🚨 Prerequisites

Before you dive in, make sure you have these tools ready:
- **jq**: A lightweight and flexible command-line JSON processor.
- **Docker Compose**: To set up and manage your Metabase service with a PostgreSQL backend.

Here’s a sample `docker-compose.yml` to get you started:

```yaml
services:
  metabase:
    image: metabase/metabase:latest
    container_name: metabase
    hostname: metabase
    ports:
      - "3000:3000"
    environment:
      MB_DB_TYPE: "postgres"
      MB_DB_DBNAME: "metabase_db"
      MB_DB_PORT: "5432"
      MB_DB_USER: "metabase_user"
      MB_DB_PASS: "metabase_password"
      MB_DB_HOST: "metabase_db"
      MB_ADMIN_EMAIL: "youremail@example.com"
      MB_USER_EMAIL: "youremail@example.com"
      MB_USER_PASSWORD: "youpassword"
      MB_DISABLE_SESSION_THROTTLE: "true"
      MB_SOURCE_ADDRESS_HEADER: NULL
      MB_SHOW_LIGHTHOUSE_ILLUSTRATION: "false"
      MB_NO_SURVEYS: "true"
      MB_LOAD_ANALYTICS_CONTENT: "false"
      MB_ANON_TRACKING_ENABLED: "false"
      MB_SETUP_TOKEN: ""
    volumes:
      - metabase_data:/metabase-data
    depends_on:
      - metabase_db

  metabase_db:
    image: postgres:16.3
    container_name: "metabase_db"
    hostname: "metabase_db"
    ports:
      - "5858:5432"
    environment:
      POSTGRES_DB: "metabase_db"
      POSTGRES_USER: "metabase_user"
      POSTGRES_PASSWORD: "metabase_password"
    volumes:
      - metabase_db_data:/var/lib/postgresql/data

volumes:
  metabase_data:
  metabase_db_data:
```

---

## 📝 Script Breakdown

### 1. Configure Your Environment

Start by setting up your script with essential variables. Tailor these to match your Metabase setup and Docker network:

```bash
METABASE_URL="http://localhost:3000/api"
METABASE_USERNAME="something@example.com"
METABASE_PASSWORD="yourpassword"

NETWORK_NAME="your_private_network_name"
```

### 2. Find the Docker Network Gateway IP

The script will automatically retrieve the gateway IP for your Docker network. If it can't find one, it will default to `127.0.0.1`:

```bash
GATEWAY_IP=$(docker network inspect "$NETWORK_NAME" --format '{{range .IPAM.Config}}{{.Gateway}}{{end}}')
if [ -z "$GATEWAY_IP" ]; then
  GATEWAY_IP="127.0.0.1"
  echo "Gateway IP for network $NETWORK_NAME not found. Using default IP $GATEWAY_IP."
else
  echo "Gateway IP for network $NETWORK_NAME is $GATEWAY_IP"
fi
```

### 3. Define Your Database Configurations

Add the PostgreSQL databases you wish to integrate into the `DATABASES` array. The format is:

```
"NAME|NETWORK_IP|PORT|DB_NAME|USER|PASSWORD"
```

For example:

```bash
DATABASES=(
  "User Database|$GATEWAY_IP|5438|user_database|user_user|user_password"
  # Add more databases here
)
```

### 4. Authenticate with Metabase

This function logs into your Metabase instance and grabs a session token:

```bash
authenticate_metabase() {
  echo "Authenticating with Metabase..."
  SESSION_ID=$(curl -s -X POST -H "Content-Type: application/json"     -d "{"username": "${METABASE_USERNAME}", "password": "${METABASE_PASSWORD}"}"     "${METABASE_URL}/session" | jq -r '.id')

  if [ -z "$SESSION_ID" ]; then
    echo "Authentication failed. Exiting."
    exit 1
  fi

  echo "Authenticated successfully. Session ID: ${SESSION_ID}"
}
```

### 5. Add Your Databases to Metabase

This function sends a request to the Metabase API to add each database:

```bash
add_database() {
  local NAME=$1
  local HOST=$2
  local PORT=$3
  local DBNAME=$4
  local USER=$5
  local PASSWORD=$6

  echo "Adding database: $NAME..."
  
  RESPONSE=$(curl -s -X POST -H "Content-Type: application/json" -H "X-Metabase-Session: ${SESSION_ID}"     -d "{
      "name": "${NAME}",
      "engine": "postgres",
      "details": {
        "host": "${HOST}",
        "port": ${PORT},
        "dbname": "${DBNAME}",
        "user": "${USER}",
        "password": "${PASSWORD}"
      }
    }"     "${METABASE_URL}/database")

  if echo "$RESPONSE" | jq -e '.id' >/dev/null; then
    echo "Database '$NAME' added successfully."
  else
    echo "Failed to add database '$NAME'. Response: $RESPONSE"
  fi
}
```

### 6. Run the Script

Authenticate with Metabase, and then add your databases:

```bash
authenticate_metabase

for db in "${DATABASES[@]}"; do
  IFS='|' read -r DB_NAME DB_HOST DB_PORT DB_DBNAME DB_USER DB_PASSWORD <<< "$db"
  add_database "$DB_NAME" "$DB_HOST" "$DB_PORT" "$DB_DBNAME" "$DB_USER" "$DB_PASSWORD"
done

echo "Finished adding databases."
```

## 🚀 How to Use

1. **Set Up a Metabase User**: Make sure you have a user account ready in your Metabase instance.

2. **Customize the Script**: Edit the configuration variables to match your Metabase setup and Docker network. Populate the `DATABASES` array with your PostgreSQL databases.

3. **Execute the Script**: Make the script executable and run it:

    ```bash
    chmod +x add_datasources.sh
    ./add_datasources.sh
    ```

4. **Verify the Results**: Log into your Metabase instance and check if your databases have been successfully added.

---

## 📜 License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for more details.
