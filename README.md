# Guide to Deploying Convex Locally with PostgreSQL

## Introduction

I initially tried running Convex in Docker, but in my case, Convex couldn't connect from the Docker container to an external database. So, I decided to deploy everything locally. 

To simplify the local setup, I launched Convex in Docker and connected to the backend container when needed—for example, to generate admin keys.

## Step 1: Clone the Repository

First, clone the `convex-backend` repository:
```sh
git clone https://github.com/get-convex/convex-backend.git
cd convex-backend/self-hosted/docker-build
```

## Step 2: Download the Precompiled Backend

Download the precompiled local build of Convex from GitHub:
```sh
wget https://github.com/get-convex/convex-backend/releases/download/precompiled-2025-02-14-3ed7cf0/convex-local-backend-aarch64-unknown-linux-gnu.zip
```

Unzip the archive:
```sh
unzip convex-local-backend-aarch64-unknown-linux-gnu.zip
```

You should now have the `convex-local-backend` executable in the same directory.

## Step 3: Set Up the Environment Variables

Create a `.env` file in the same directory:
```env
DATABASE_URL=postgresql://your_db_user:your_db_password@your_db_host
INSTANCE_NAME=convex
CONVEX_CLOUD_ORIGIN=http://your-domain-or-ip:3210
CONVEX_SITE_ORIGIN=http://your-domain-or-ip:3211
DISABLE_BEACON=true
REDACT_LOGS_TO_CLIENT=false
DATA_DIR=/convex/data
RUST_LOG=info,common::errors=debug
RUST_BACKTRACE=1
```
**Important:** The `INSTANCE_NAME` field can be any value but must not contain spaces.

### Generate an `INSTANCE_SECRET`
Since we do not store `INSTANCE_SECRET` in the `.env` file for security reasons, generate it manually:
```sh
openssl rand -hex 32
```
Copy the generated secret and use it when required.

## Step 4: Prepare PostgreSQL

Convex requires a PostgreSQL database named `convex`. The database connection string in `DATABASE_URL` should point to the PostgreSQL server but **should not include the database name** because Convex automatically connects to the `convex` database.

If using **NeonDB**, create the database on their platform.
If hosting **your own PostgreSQL**, you need to:
- Run PostgreSQL in Docker.
- Enable `sslmode=true` and set up TLS.
- If TLS is not configured, Convex won't be production-ready.

To disable SSL for local testing, you can add `--do-not-require-ssl` in the startup script.

## Step 5: Modify the Startup Script

Edit `run_backend.sh` as follows:
```sh
#!/bin/bash

# Set data directories
export DATA_DIR=${DATA_DIR:-/convex/data}
export TMPDIR=${TMPDIR:-"$DATA_DIR/tmp"}
export STORAGE_DIR=${STORAGE_DIR:-"$DATA_DIR/storage"}

set -e
mkdir -p "$TMPDIR" "$STORAGE_DIR"

# Load environment variables
if [ -f .env ]; then
    set -a
    source .env
    set +a
fi

# Check if DATABASE_URL is set
if [ -z "$DATABASE_URL" ]; then
    echo "Error: DATABASE_URL is not set. Define it in the .env file."
    exit 1
fi

# Set PostgreSQL parameters
DB_FLAGS=(--db postgres-v5 "$DATABASE_URL")

exec ./convex-local-backend "$@" \
    --instance-name "$INSTANCE_NAME" \
    --instance-secret "$INSTANCE_SECRET" \
    --local-storage "$STORAGE_DIR" \
    --port 3210 \
    --site-proxy-port 3211 \
    --convex-origin "$CONVEX_CLOUD_ORIGIN" \
    --convex-site "$CONVEX_SITE_ORIGIN" \
    --beacon-tag "self-hosted-docker" \
    ${DISABLE_BEACON:+--disable-beacon} \
    ${REDACT_LOGS_TO_CLIENT:+--redact-logs-to-client} \
    ${DB_FLAGS[@]} \
    --do-not-require-ssl # DON'T USE IN PRODUCTION
```

## Step 6: Running Convex Backend

I use `tmux` to keep Convex running outside of Docker:
```sh
tmux new -s backend
./run_backend.sh
```

## Step 7: Launch the Convex Dashboard in Docker

The Convex backend in Docker is only needed for generating the admin key. Once you have the key, you can stop it.

To temporarily run Convex in Docker, go to the `/self-hosted/docker/` directory and follow the official guide.

To start the Convex dashboard, use this `docker-compose.yml`:
```yaml
services:
  dashboard:
    image: ghcr.io/get-convex/convex-dashboard:4499dd4fd7f2148687a7774599c613d052950f46
    ports:
      - "6791:6791"
    environment:
      - NEXT_PUBLIC_DEPLOYMENT_URL=http://your-domain-or-ip:3210
```

Once running, access the dashboard at `http://your-domain-or-ip:6791`.

## Step 8: Generate the Admin Key

To access the dashboard, you need an admin key. Connect to the Convex backend container and run the following command:

```sh
ls
```
This should output:
```sh
convex-local-backend  generate_admin_key.sh  read_credentials.sh
data                  generate_key           run_backend.sh
```
Modify `generate_admin_key.sh`:
```sh
#!/bin/bash
set -e

NAME="your_instance_name"
SECRET="your_instance_secret"

ADMIN_KEY=$(./generate_key "$NAME" "$SECRET")

echo "$ADMIN_KEY"
```
Replace `your_instance_name` and `your_instance_secret` with the values from `.env`.

Now, run:
```sh
./generate_admin_key.sh
```
The output will look like this:
```sh
Admin key:
convex|01c0356606ef6023bf805f2928e5aeac82d6ebe05e84dddb1d0b74b6ab6679c9b97e64e8f7e5dfe2ec855d3b9415991955
```
Copy this key and use it to log into the dashboard.

## Conclusion

Now you have a fully functional local deployment of Convex with PostgreSQL, a dashboard, and admin access. For production, use a domain instead of an IP address and properly configure PostgreSQL with SSL.

