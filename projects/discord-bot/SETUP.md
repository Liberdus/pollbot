# Pollbot Setup Guide

This document walks through provisioning a fresh server, configuring Firebase and Discord, and running [pollbot](https://github.com/qntnt/pollbot).

## 1. Clone & Install Tooling

```bash
git clone https://github.com/qntnt/pollbot.git
sudo apt-get update
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs make protobuf-compiler
sudo corepack enable                     # enables Yarn 1.x
curl -sSL https://github.com/bufbuild/buf/releases/latest/download/buf-Linux-x86_64.tar.gz -o /tmp/buf.tar.gz
tar -xzf /tmp/buf.tar.gz -C /tmp
sudo mv /tmp/buf/bin/buf /usr/local/bin/buf
```

## 2. Firebase / Firestore Preparation

1. Visit the [Firebase Console](https://console.firebase.google.com/) and create or select a project.
2. Build → **Firestore Database** → create a database in **Production** mode.
3. Project settings → **Service Accounts** → `Generate new private key` (JSON). Copy the file to the server, e.g.
   ```bash
   mkdir -p /root/keys
   mv ~/Downloads/<downloaded-key>.json /root/keys/pollbot-service-account.json
   chmod 600 /root/keys/pollbot-service-account.json
   ```
4. In Firestore’s data tab create the following top-level collections: `polls`, `ballots`, `guilds`, `counters`.
5. Under `counters`, add a document `poll_id` with `{ value: 0 }`.
6. Ensure each document in `polls` has a `features` field of type **array** (set it to `[]` for legacy docs).

## 3. Discord Bot Registration

1. Open the [Discord Developer Portal](https://discord.com/developers/applications), create an application, and add a bot user.
2. Record the bot token (`DISCORD_TOKEN`) and application ID (`DISCORD_CLIENT_ID`).
3. OAuth2 → **URL Generator**: select `bot` and `applications.commands`, choose permissions (Send Messages, Read Message History, Add Reactions), and invite the bot to your server.

Users must allow direct messages from server members (Server → Privacy Settings) so the ballot button can DM them.

## 4. Generate the TypeScript IDL Package

```bash
cd /root/pollbot/projects/idl
buf mod update
make build
# If the build fails because ts-proto is missing:
cd ../../packages/idl-ts && yarn install
cd ../../projects/idl && make build
```

## 5. Install Discord Bot Dependencies

```bash
cd /root/pollbot/projects/discord-bot
yarn add --exact discord.js@13.6.0 @discordjs/builders@0.11.0 discord-api-types@0.26.1 @discordjs/rest@0.3.0
yarn install
```

## 6. Environment Variables

Export the required environment variables in your shell (or add them to your profile/startup script):

```bash
export DISCORD_TOKEN="your-token"
export DISCORD_CLIENT_ID="your-application-id"
export GOOGLE_APPLICATION_CREDENTIALS="/root/keys/pollbot-service-account.json"
export NODE_ENV="staging"   # optional
```

Run these in every new shell before launching the bot (or source a script that contains them).

## 7. Run Pollbot

Development (hot reload):

```bash
cd /root/pollbot/projects/discord-bot
yarn dev
```

Production-style:

```bash
yarn build
node build/index.js
```

Slash commands are registered on startup; global propagation can take up to an hour.

## 8. Smoke Test

1. Run `/poll_create` in your server and confirm a poll message appears.
2. Click **Request Ballot** (ensure DMs are enabled) and verify a ballot is delivered via DM.
3. Use `/poll_results` or `/poll_close` after submitting ballots to validate Firestore writes.

## 9. Maintenance Tips

- Re-run `make build` whenever protobuf definitions change.
- Keep the Discord packages pinned unless you update the code for newer APIs.
- Rotate service-account keys and Discord tokens if they leak.
- Backfill any imported poll documents so `features` is always an array.
