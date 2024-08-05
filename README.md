# Metabase Automatic Docker Integration Script

This repository contains a Bash script to automate the process of adding PostgreSQL databases to a Metabase Docker container. The script interacts with the Metabase API to configure databases, making it easier to manage and integrate various data sources into your Metabase instance.

## Overview

The script performs the following tasks:
1. Retrieves the gateway IP address of a specified Docker network.
2. Authenticates with the Metabase API to obtain a session token.
3. Adds PostgreSQL databases to the Metabase instance using the session token.

## Prerequisites

Before running the script, ensure you have the following:
- **Docker**: To manage your Docker containers and networks.
- **jq**: A command-line tool for parsing JSON, used in the script for handling API responses.

## Script Breakdown

### 1. Configuration

The script starts by defining necessary variables for Metabase API access and Docker network details. Customize these variables according to your setup:

```bash
METABASE_URL="http://localhost:3000/api"
METABASE_USERNAME="something@example.com"
METABASE_PASSWORD="yourpassword"

NETWORK_NAME="your_private_network_name"
```

- `METABASE_URL`: The URL of your Metabase API.
- `METABASE_USERNAME` and `METABASE_PASSWORD`: Credentials for Metabase authentication.
- `NETWORK_NAME`: The name of the Docker network to retrieve the gateway IP from.

### 2. Retrieve Docker Network Gateway IP

The script fetches the gateway IP for the specified Docker network. If the gateway IP is not found, it defaults to `127.0.0.1`.

```bash
GATEWAY_IP=$(docker network inspect "$NETWORK_NAME" --format '{{range .IPAM.Config}}{{.Gateway}}{{end}}')
if [ -z "$GATEWAY_IP" ]; then
  GATEWAY_IP="127.0.0.1"
  echo "Gateway IP for network $NETWORK_NAME not found. Using default IP $GATEWAY_IP."
else
  echo "Gateway IP for network $NETWORK_NAME is $GATEWAY_IP"
fi
```

### 3. Define Database Configurations

Specify the PostgreSQL databases you want to add to Metabase in the `DATABASES` array. Each entry follows the format:

```
"NAME|NETWORK_IP|PORT|DB_NAME|USER|PASSWORD"
```

Example:

```bash
DATABASES=(
  "User Database|$GATEWAY_IP|5438|user_database|user_user|user_password"
  # Add more databases here
)
```

### 4. Authenticate with Metabase

The `authenticate_metabase` function handles authentication with the Metabase API and retrieves a session token.

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

### 5. Add Databases to Metabase

The `add_database` function sends a request to add each PostgreSQL database to Metabase using the previously obtained session token.

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

### 6. Execution

The script first authenticates with Metabase and then iterates over the database configurations to add each database.

```bash
authenticate_metabase

for db in "${DATABASES[@]}"; do
  IFS='|' read -r DB_NAME DB_HOST DB_PORT DB_DBNAME DB_USER DB_PASSWORD <<< "$db"
  add_database "$DB_NAME" "$DB_HOST" "$DB_PORT" "$DB_DBNAME" "$DB_USER" "$DB_PASSWORD"
done

echo "Finished adding databases."
```

## Usage
1. **Create a user in metabase**

2. **Customize the Script**: Update the configuration variables with your Metabase details and Docker network name. Modify the `DATABASES` array to include your PostgreSQL databases.

3. **Run the Script**: Make the script executable and run it:

    ```bash
    chmod +x add_databases.sh
    ./add_databases.sh
    ```

4. **Check Results**: Verify that the databases have been added to your Metabase instance by logging into Metabase and checking the list of databases.

![image](https://github.com/user-attachments/assets/5cc12efc-6123-4d22-9aa1-6373bac8bf81)


## Contributing

Feel free to fork this repository, make improvements, and submit pull requests. If you encounter any issues or have suggestions, please open an issue in the GitHub repository.

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

---
