# Immich Transfer Tool

```
╔══════════════════════════════════════════════════════════╗
║                   IMMICH TRANSFER TOOL                   ║
║          Transfer photos between users safely            ║
║                                                          ║
║            github.com/pedrorj2/immich-transfer           ║
╚══════════════════════════════════════════════════════════╝
```

Transfer photos between users in Immich, including faces and associated metadata — without re-scanning.

---

> [!CAUTION]
> **This tool directly modifies your Immich database.**
> 
> It was created out of personal necessity and worked for me, but **I am not responsible for any data loss or database corruption.**
> 
> **ALWAYS make a backup before using this tool. Use at your own risk.**

---

## Problem it solves

Immich doesn't allow transferring photo ownership between users from the UI. This tool allows:

- ✅ Transfer photos from an album to another user
- ✅ Correctly handle associated faces/persons
- ✅ No need to re-scan faces (keeps embeddings)
- ✅ Interactive CLI with dry-run mode
- ✅ Automatic configuration setup

> [!NOTE]
> **How faces are handled:**
> 
> Imagine you have 1,000,000 photos and want to transfer an album with 10,000 photos.
> 
> | Scenario | Action | Result |
> |----------|--------|--------|
> | Person **only appears** in the 10,000 photos being transferred | **TRANSFER** | Person is moved to new user, removed from original user (no orphan data) |
> | Person appears in **both** the 10,000 photos AND other photos | **DUPLICATE** | Person is copied to new user, original stays intact for remaining photos |
> 
> This ensures no orphan face data remains in your database, and persons that still have photos in your library are preserved.

---

## Requirements

- Python 3.11+
- Access to Immich API (API Key)
- Direct access to Immich's PostgreSQL database

---

## Installation

```bash
git clone https://github.com/pedrorj2/immich-transfer.git
cd immich-transfer
pip install -r requirements.txt
```

---

## Backup First!

> [!WARNING]
> **ALWAYS backup your database before running this tool!**
> 
> ```bash
> docker exec -t immich_postgres pg_dumpall -c -U immich > backup_$(date +%Y%m%d_%H%M%S).sql
> ```
> 
> To restore if something goes wrong:
> ```bash
> docker exec -i immich_postgres psql -U immich -d immichdb < backup_XXXXX.sql
> ```

---

## Usage

```bash
python transfer.py
```

On first run, the tool will guide you through configuration:

```
╔══════════════════════════════════════════════════════════╗
║                   IMMICH TRANSFER TOOL                   ║
║          Transfer photos between users safely            ║
║                                                          ║
║            github.com/pedrorj2/immich-transfer           ║
╚══════════════════════════════════════════════════════════╝

  ⚠ Configuration not found or incomplete.
  Run setup now? (y/n): y
```

### Main Menu

```
┌─────────────────────────────────────┐
│             MAIN MENU               │
├─────────────────────────────────────┤
│  1.  List users                     │
│  2.  List albums                    │
│  3.  Transfer album (dry run)       │
│  4.  Transfer album (EXECUTE)       │
├─────────────────────────────────────┤
│  5.  Repair faces (dry run)         │
│  6.  Repair faces (EXECUTE)         │
├─────────────────────────────────────┤
│  7.  Reconfigure                    │
│  8.  Clear config                   │
│  0.  Exit                           │
└─────────────────────────────────────┘
```

### Repair Faces

Use options 5-6 if you transferred photos **manually** (without this tool) and faces are broken. This will:
1. Find faces linked to the wrong user's persons
2. Duplicate those persons for the correct user
3. Re-link the faces to the new persons
4. Attempt to fix broken profile pictures

---

## Recommended Workflow

> [!TIP]
> **Always test with dry-run mode first!**
> 
> 1. **Make a backup** (see above)
> 2. **Dry run first** (option 3) → select source user, album, and destination user interactively
> 3. **Execute transfer** (option 4) → only if dry run looks correct

The tool will guide you step by step:
1. Select the source user (who owns the album)
2. Select the album to transfer
3. Select the destination user
4. Confirm and execute

### What is Dry Run?

> [!NOTE]
> **Dry run simulates the transfer without making any changes to the database.**
> 
> It shows you:
> - How many assets would be transferred
> - Which persons would be **transferred** (exclusive to album)
> - Which persons would be **duplicated** (appear in other photos too)
> 
> Use this to verify everything looks correct before running the actual transfer.

---

## Configuration

The tool creates `config.py` automatically during setup. You can also edit it manually:

```python
# Immich API URL
API_URL = "http://your-server:2283/api"

# API Key with admin permissions
API_KEY = "your-api-key"

# PostgreSQL connection
DB_CONFIG = {
    "host": "your-server",
    "port": 5432,
    "database": "immichdb",
    "user": "immich",
    "password": "your-password"
}
```

### Getting API Key

1. In Immich → Profile → Account Settings → API Keys
2. Create a new API Key
3. Copy the key (only shown once)

### PostgreSQL Connection

> [!IMPORTANT]
> By default, PostgreSQL in Immich **is not exposed** outside Docker. You need to temporarily expose the port.

#### Step 1: Expose PostgreSQL port

Find your Immich's `docker-compose.yml`:

```bash
# Usually in /opt/immich/ or check with:
docker inspect immich_server | grep "compose.project.config_files"
```

Edit `docker-compose.yml` and add `ports` to the database service:

```yaml
database:
  container_name: immich_postgres
  # ... other config ...
  ports:
    - "5432:5432"
```

Restart Immich:

```bash
cd /opt/immich  # or your immich directory
docker compose up -d
```

#### Step 2: Run the tool from your PC

```bash
python transfer.py
```

Use your server's IP as the database host (e.g., `192.168.1.6`).

#### Step 3: Close the port after use (recommended)

Once you're done transferring, remove or comment out the `ports` section and restart:

```yaml
database:
  container_name: immich_postgres
  # ... other config ...
  # ports:
  #   - "5432:5432"
```

```bash
docker compose up -d
```

> [!NOTE]
> If you're on a trusted local network, leaving the port open is not a major security risk. But it's good practice to close it when not needed.

### Finding PostgreSQL credentials

Check the `.env` file in your Immich directory:

```bash
cat /opt/immich/.env | grep DB_
```

You'll see:
- `DB_PASSWORD` - Database password
- `DB_USERNAME` - Database user (usually `immich`)
- `DB_DATABASE_NAME` - Database name (usually `immichdb`)

---

## How it works

### Asset Transfer

Updates the `ownerId` field in the `asset` table for all album assets.

### Face Handling

The tool analyzes each person that appears in the album:

```
┌─────────────────────────────────────────────────────────────┐
│  For each person detected in the album:                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Q: Does this person have faces in OTHER photos             │
│     (outside this album)?                                   │
│                                                             │
│     NO  → TRANSFER: Move person to new user                 │
│           The person is removed from original user          │
│           (prevents orphan face data)                       │
│                                                             │
│     YES → DUPLICATE: Copy person to new user                │
│           Original person stays for remaining photos        │
│           Only faces from transferred photos are re-linked  │
│                                                             │
│  Face embeddings (face_search) stay linked to faces         │
│  No re-scanning needed!                                     │
└─────────────────────────────────────────────────────────────┘
```

> [!TIP]
> **Example:** Your mom appears in 500 photos across your library. You're transferring an album with 100 of those photos.
> - Mom has 400 photos **outside** the album → **DUPLICATE**
> - Mom will exist in both users' face libraries
> - Your library keeps mom with 400 photos, new user gets mom with 100 photos

### Example Output

```
┌────────────────────────────────────────────────────┐
│              ALBUM TRANSFER ANALYSIS               │
└────────────────────────────────────────────────────┘

  Assets to transfer: 9905
  Persons detected: 45

  Person analysis:
    → John Doe (234 faces) [TRANSFER]
    → Jane Smith (89 in album, 156 outside) [DUPLICATE]
    → Unknown (12 faces) [TRANSFER]

┌────────────────────────────────────────────────────┐
│              TRANSFER COMPLETED                    │
└────────────────────────────────────────────────────┘

  Summary:
    • Assets transferred:  9905
    • Persons transferred: 32
    • Persons duplicated:  13
```

---

## Tested On

| Component | Version |
|-----------|---------|
| Immich | v2.3.1 |
| PostgreSQL | 14 |
| Python | 3.11+ |

---

## Limitations

- Only transfers by album (not by tag — could be added)
- Requires direct database access
- Doesn't move physical files (only changes ownership in DB)
- Album ownership is not transferred (only the assets inside)

---

## Known Issues

> [!WARNING]
> **After transfer, duplicated persons are independent.**
> 
> When a person is duplicated (appears in both users' photos), they become **separate persons** in each user's library. This means:
> - "Mom" in User A and "Mom" in User B are **different** persons
> - Changes to one don't affect the other
> - You may need to manually set profile pictures for duplicated persons

> [!NOTE]
> **Profile pictures for duplicated persons may need to be set manually.**
> 
> The tool attempts to set a profile picture automatically, but in some cases you may need to select a face thumbnail manually in Immich's People section.

---

## Contributing

PRs welcome! Some ideas:

- [ ] Transfer by tag
- [ ] Web interface
- [ ] Rollback functionality
- [ ] Progress bar for large transfers

---

## License

MIT

---

## Author

Created by **[@pedrorj2](https://github.com/pedrorj2)**


> [!NOTE]
> **Remember: Always backup before using. Test with dry-run. Use at your own risk.**
