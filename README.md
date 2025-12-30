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

- Python 3.8+
- Access to Immich API (admin API Key)
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
│  5.  Reconfigure                    │
│  6.  Clear config                   │
│  0.  Exit                           │
└─────────────────────────────────────┘
```

---

## Recommended Workflow

> [!TIP]
> **Always test with dry-run mode first!**
> 
> 1. **Make a backup** (see above)
> 2. **List albums** (option 2) → get the album ID
> 3. **List users** (option 1) → get the destination user ID  
> 4. **Dry run first** (option 3) → see what will be transferred
> 5. **Execute transfer** (option 4) → only if dry run looks correct

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
> By default, PostgreSQL in Immich **is not exposed** outside Docker.
> 
> You have two options:

#### Option A: Run from Immich server ✅ (recommended)

```bash
# From your PC, copy files to server
scp -r immich-transfer/ user@your-server:/root/

# On the server
ssh user@your-server
cd /root/immich-transfer
pip install -r requirements.txt
python transfer.py
```

In `config.py` use the container hostname:
```python
DB_CONFIG = {
    "host": "immich_postgres",
    ...
}
```

#### Option B: Expose PostgreSQL port ⚠️ (less secure)

Edit your `docker-compose.yml`:

```yaml
immich_postgres:
  ports:
    - "5432:5432"
```

Then restart: `docker compose up -d`

> [!CAUTION]
> This exposes your database to the local network. Only do this if you trust your network.

### Finding PostgreSQL password

```bash
docker exec -t immich_postgres env | grep POSTGRES_PASSWORD
```

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
| Python | 3.8+ |

---

## Limitations

- Only transfers by album (not by tag — could be added)
- Requires direct database access
- Doesn't move physical files (only changes ownership in DB)

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
