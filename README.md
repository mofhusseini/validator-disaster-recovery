# Participant Recovery Guide

This guide walks you through the process of recovering a participant from an existing database backup to a new participant setup.

## Prerequisites

- Access to a PostgreSQL database
- Docker installed and running
- Canton participant database backup

## Step 1: Restore Database Backup

Restore your database backup into a fresh PostgreSQL database. Ensure you have proper access credentials.

**Key Details:**

- Target database: `participant` database (name depends on migration ID)
- Example database name: `participant_2`
- Default username: `cnadmin`

## Step 2: Run the Participant

Create a `docker-compose.yml` file with the following configuration:

```yaml
name: splice-validator

services:
  participant:
    image: "ghcr.io/digital-asset/decentralized-canton-sync/docker/canton-participant:0.3.21"
    ports:
      - 5001:5001
      - 5002:5002
    environment:
      - CANTON_PARTICIPANT_POSTGRES_SERVER=host.docker.internal
      - CANTON_PARTICIPANT_POSTGRES_SCHEMA=participant
      - CANTON_PARTICIPANT_POSTGRES_USER=cnadmin
      - CANTON_PARTICIPANT_POSTGRES_PASSWORD=PASSSSSSSS
      - CANTON_PARTICIPANT_POSTGRES_DB=participant_2
      - CANTON_PARTICIPANT_POSTGRES_PORT=5435
      - CANTON_PARTICIPANT_ADMIN_USER_NAME=ledger-api-user
      - ADDITIONAL_CONFIG_DISABLE_AUTH=canton.participants.participant.ledger-api.auth-services=[]
      - ADDITIONAL_CONFIG_ALLOW_INSECURE=canton.participants.participant.http-ledger-api.allow-insecure-tokens=true
    restart: always
```

Start the participant:

```bash
docker-compose up -d
```

## Step 3: Create Console Configuration

Create a `console.conf` file:

```hocon
canton {
  remote-participants {
    participant {
      admin-api {
        port = 5002
        address = host.docker.internal
      }
      ledger-api {
        port = 5001
        address = host.docker.internal
      }
      token = ""
    }
  }
  features.enable-preview-commands = yes
  features.enable-testing-commands = yes
  features.enable-repair-commands = yes
}
```

## Step 4: Launch Canton Console

Run the Canton console using Docker:

```bash
docker run -it --rm --network host \
  -v $(pwd)/console.conf:/app/app.conf \
  -v $(PWD)/shared:/shared \
  --entrypoint bash \
  ghcr.io/digital-asset/decentralized-canton-sync/docker/canton:0.3.12
```

## Step 5: Extract Keys and Authorized Store Snapshot

Inside the Canton console, execute the following script to extract the necessary data:

```scala
import java.util.Base64
import java.io._

val keys = "[" + participant.keys.secret.list().filter(k => k.name.get.unwrap != "cometbft-governance-keys").map(key => s"{\"keyPair\": \"${Base64.getEncoder.encodeToString(participant.keys.secret.download(key.publicKey.fingerprint).toByteArray)}\", \"name\": \"${key.name.get.unwrap}\"}").mkString(",") + "]"

val authorizedStoreSnapshot = Base64.getEncoder.encodeToString(participant.topology.transactions.export_topology_snapshot(filterStore = "Authorized", timeQuery = TimeQuery.Range(from = None, until = None), filterMappings = Seq(
  TopologyMapping.Code.NamespaceDelegation,
  TopologyMapping.Code.OwnerToKeyMapping,
  TopologyMapping.Code.IdentifierDelegation,
  TopologyMapping.Code.VettedPackages,
), filterNamespace = participant.id.namespace.toProtoPrimitive).toByteArray)

val keysFilePath = "/shared/keys.json"
val writer = new BufferedWriter(new FileWriter(keysFilePath))
try {
  writer.write(keys)
} finally {
  writer.close()
}

val authorizedStoreSnapshotFilePath = "/shared/authorizedStoreSnapshot.txt"
val writer2 = new BufferedWriter(new FileWriter(authorizedStoreSnapshotFilePath))
try {
  writer2.write(authorizedStoreSnapshot)
} finally {
  writer2.close()
}
```

This will generate:

- `./shared/keys.json` - Contains the extracted keys
- `./shared/authorizedStoreSnapshot.txt` - Contains the authorized store snapshot

## Step 6: Create Identities File

Construct an `identities.json` file with the following structure:

```json
{
  "id": "PAR::<party_hint>::<fingerprint_of_public_key>",
  "keys": [
    {
      "keyPair": "Cp8BCpwBCikQA.....",
      "name": "namespace"
    },
    {
      "keyPair": "CqUBCqIB....",
      "name": "signing"
    },
    {
      "keyPair": "CswCEskCCmEQAhp.....",
      "name": "encryption"
    }
  ],
  "authorizedStoreSnapshot": "CuLeBQqyAgoLC.....",
  "version": "0.1.0"
}
```

Use the data from the files generated in Step 5 to populate the `keys` array and `authorizedStoreSnapshot` field.

## Step 7: Run Migration Script

Execute the migration script with your specific configuration:

```bash
#!/bin/bash

# Configuration - Update these values for your environment
SCAN_URL=https://scan.sv-2.global.canton.network.TBD  # TODO: Update with actual URL
party_hint="current_party_hint"  # TODO: Set your party hint
party_id="$party_hint::current_fingerprint"  # TODO: Update with actual fingerprint
MIGRATION_ID=3  # TODO: Set current migration ID
SPONSOR_SV_URL=https://sv.sv-2.global.canton.network.TBD  # TODO: Update with actual URL
NEW_PARTICIPANT_NAME="new-participant-id"  # TODO: Set new participant ID

# Get domain ID
domain_id=$(curl -fsS "$SCAN_URL/api/scan/v0/dso" | jq -r '.dso_rules.domain_id')

# Get participant ID
participant_id=$(
  curl -fsS "$SCAN_URL/api/scan/v0/domains/$domain_id/parties/$party_id/participant-id" |
    jq -r .participant_id
)

# Update identities file with participant ID
cat ./identities.json |
  jq --arg participant_id "$participant_id" '.id = $participant_id' > ./identities.updated.json

# Run migration
echo "Migrating $party_hint from $participant_id to $NEW_PARTICIPANT_NAME"
./start.sh -w -s "$SPONSOR_SV_URL" -o "" -p "$party_hint" -m "$MIGRATION_ID" -P "$NEW_PARTICIPANT_NAME" -i ./identities.updated.json
```

## Important Notes

- Replace all `TODO` placeholders with your actual values
- Ensure all URLs, database credentials, and participant names are correctly configured
- The `shared` directory should be writable by the Docker container
- Keep your database credentials secure and never commit them to version control

## Troubleshooting

- Verify database connectivity before starting the participant
- Check Docker logs if the participant fails to start
- Ensure all required ports are available and not blocked by firewall rules
- Validate JSON syntax in configuration files before use
