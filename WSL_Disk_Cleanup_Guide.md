# Reclaiming C: disk space from WSL (Ubuntu + MySQL in Docker)

## Problem

- A very large SELECT ran in a MySQL Docker container inside WSL (`Ubuntu`) and filled the disk.
- Deleting databases inside WSL did not free space on Windows. C: showed only about 10 GB free.
- `wsl --shutdown` did not keep WSL stopped. It kept starting again.

## Why it happens

WSL keeps each distro inside a virtual disk file (`ext4.vhdx`). That file **grows automatically but never shrinks**
when you delete files inside Linux. To get the space back on Windows you must:

1. Free space inside Linux.
2. Trim it (`fstrim`) so the free blocks are marked as unused.
3. Shut down WSL and **compact** the `.vhdx`.

---

## Step 0: Stop WSL from restarting

The cause was a **Kiro window with a WSL folder open**. It reconnects and restarts the `Ubuntu` distro.

- Close any Kiro / VS Code window connected to WSL (or use *File → Close Remote Connection*).
- Close WSL terminals.
- Then, in PowerShell:

```powershell
wsl --shutdown
wsl --list --verbose   # both distros must show "Stopped"
```

> The `vmmemWSL` process can stay up for a short time after the distros stop. That is fine.

## Step 1: Find the virtual disk files (PowerShell)

```powershell
Get-ChildItem "$env:LOCALAPPDATA\Packages\CanonicalGroupLimited.Ubuntu*\LocalState\ext4.vhdx" |
  ForEach-Object { "{0,8:N2} GB  {1}" -f ($_.Length/1GB), $_.FullName }
```

| Distro        | Path | Size before |
|---------------|------|-------------|
| Ubuntu        | `C:\Users\<YourUser>\AppData\Local\Packages\CanonicalGroupLimited.Ubuntu_79rhkp1fndgsc\LocalState\ext4.vhdx` | **257.43 GB** |
| Ubuntu-24.04  | `C:\Users\<YourUser>\AppData\Local\Packages\CanonicalGroupLimited.Ubuntu24.04LTS_79rhkp1fndgsc\LocalState\ext4.vhdx` | 2.19 GB |

## Step 2: Find what uses the space inside WSL

```bash
df -h /                         # real usage inside Linux (was 199G used)
docker system df                # Docker images / containers / volumes / cache
sudo du -xh / --max-depth=3 2>/dev/null | sort -h | tail -20
```

Results:

- `~/Documentos` → **138G**
- `~/Downloads` → 28G
- `/var/lib/docker` → 16G (about 11 GB of unused images + 760 MB build cache)
- Docker volumes were only 281 MB, so the MySQL data was in a **bind-mounted folder** inside `Documentos`.

Look 2 levels deeper:

```bash
sudo du -xh ~/Documentos --max-depth=3 2>/dev/null | sort -h | tail -30
```

> ⚠️ **Always use `sudo`.** Without it, `du` showed only 19G because the MySQL data folders are owned by
> `root` / `mysql` (UID 999). The permission errors were hidden by `2>/dev/null`.

Find folders your user cannot read:

```bash
find ~/Documentos -maxdepth 4 ! -readable -type d 2>/dev/null
```

Inspect the biggest files in the MySQL data folder:

```bash
sudo ls -lhS <mysql-data-folder> | head -20
```

Typical culprits after a huge query:

| File | What it is | How to clean |
|------|-----------|--------------|
| `binlog.000xxx` | Binary logs | In MySQL: `PURGE BINARY LOGS BEFORE NOW();` (do not `rm`) |
| `ibtmp1`, `#innodb_temp/` | Temp tablespace from the big SELECT | Restart the MySQL container |
| `*.ibd` of dropped DBs | Leftover tablespaces | Make sure the DB was really dropped |
| `undo_001`, `undo_002` | Undo logs | Can grow after long queries |

## Step 3: Clean up

```bash
docker image prune -a      # unused images (~11 GB)
docker builder prune       # build cache (~760 MB)
# docker system prune --volumes   # only if no volume holds needed data
```

Also review `~/Downloads` (28G).

## Step 4: Trim free space inside WSL

```bash
sudo fstrim -av
```

## Step 5: Shut down WSL

```powershell
wsl --shutdown
wsl --list --verbose   # confirm "Stopped"
```

## Step 6: Compact the virtual disk (PowerShell as Administrator)

Optional backup first (to a drive other than C:):

```powershell
wsl --export Ubuntu D:\ubuntu-backup.tar
```

Using `diskpart` (works on every Windows edition):

```
diskpart
select vdisk file="C:\Users\<YourUser>\AppData\Local\Packages\CanonicalGroupLimited.Ubuntu_79rhkp1fndgsc\LocalState\ext4.vhdx"
attach vdisk readonly
compact vdisk
detach vdisk
exit
```

Alternative on Windows Pro with Hyper-V:

```powershell
Optimize-VHD -Path "C:\Users\<YourUser>\AppData\Local\Packages\CanonicalGroupLimited.Ubuntu_79rhkp1fndgsc\LocalState\ext4.vhdx" -Mode Full
```

> If `attach vdisk` says the file is in use, something restarted WSL. Close it and run `wsl --shutdown` again.

---

## Result (checked 2026-11-23)

| Check | Before | After |
|-------|--------|-------|
| `Ubuntu` ext4.vhdx size | 257.43 GB | **211.39 GB** |
| C: free space | 10.77 GB | **56.76 GB** |

## Prevention (optional)

- **Sparse VHD** (`wsl --manage Ubuntu --set-sparse true`): **not enabled.** It gives space back to Windows automatically,
  but WSL warns it may risk data corruption. Repeat Steps 4–6 by hand instead.
- **Limit MySQL temp tablespace** in the container's MySQL config:
  ```ini
  innodb_temp_data_file_path = ibtmp1:12M:autoextend:max:5G
  ```
- **Expire binary logs automatically** (example: 3 days):
  ```ini
  binlog_expire_logs_seconds = 259200
  ```
