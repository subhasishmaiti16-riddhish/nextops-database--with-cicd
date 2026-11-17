# Axion PostgreSQL on Azure VM

PostgreSQL 14 running on an Ubuntu 22.04 Azure VM (`db-vm`), used as the database for the Axion telemetry system.

## Overview

| Item | Value |
|------|-------|
| Database | `axion_db` |
| App user | `dbadmin` |
| Port | `5432` |
| Client | pgAdmin 4 / psql |

## Tables

**`telemetry`**: device readings

| Column | Type |
|--------|------|
| id | uuid (auto) |
| device_id, device_type, refinery_region | varchar(50) |
| timestamp | timestamp |
| temperature, pressure, flow_rate | double precision |
| component_metrics | jsonb |
| created_at | timestamp (auto) |

**`alarms`**: device alarms

| Column | Type |
|--------|------|
| id | uuid (auto) |
| alarm_code, device_id, device_type, metric_name | varchar(50) |
| severity | varchar(20) |
| current_value, threshold_value | varchar(50) |
| message | text |
| status | varchar(20), default `ACTIVE` |
| created_at, cleared_at | timestamp |

## Remote Access Setup

1. **Azure NSG**: add an inbound rule allowing TCP `5432` from your public IP only (`/32`).
2. **`postgresql.conf`**: set `listen_addresses = '*'`.
3. **`pg_hba.conf`**: add a line for your IP:
   ```
   host    axion_db    dbadmin    <your-public-ip>/32    scram-sha-256
   ```
4. Reload PostgreSQL:
   ```bash
   sudo systemctl reload postgresql
   ```
5. Set the user password:
   ```bash
   sudo -u postgres psql -c "\password dbadmin"
   ```

## Connect with pgAdmin

- Host: `<vm-public-ip>`
- Port: `5432`
- Maintenance database: `axion_db`
- Username: `dbadmin`

Test network reachability first:

```powershell
Test-NetConnection <vm-public-ip> -Port 5432
```

## Adding Data

```sql
INSERT INTO telemetry
  (device_id, device_type, refinery_region, timestamp, temperature, pressure, flow_rate)
VALUES
  ('DEV-001', 'PUMP', 'WEST', NOW(), 85.5, 120.3, 45.2);

INSERT INTO alarms
  (alarm_code, device_id, device_type, severity, metric_name, current_value, threshold_value, message)
VALUES
  ('ALM-TEMP-HIGH', 'DEV-001', 'PUMP', 'CRITICAL', 'temperature', '98.5', '90',
   'Temperature exceeded threshold');
```

## Troubleshooting

| Symptom | Likely cause |
|---------|--------------|
| `TcpTestSucceeded : False` / timeout | Public IP changed. Update the NSG rule and `pg_hba.conf` |
| `no pg_hba.conf entry` | Missing or wrong IP, user, or database in `pg_hba.conf` |
| `password authentication failed` | Wrong or unset password. Reset with `\password` |
| `connection refused` | `listen_addresses` not set or PostgreSQL not running |
| pgAdmin `access violation` crash | pgAdmin bug. Restart, change SSL mode, reset settings, or reinstall |

## Security Notes

- Never open `5432` to `Any`. Restrict it to known IPs.
- Do not use the `postgres` superuser for remote access.
- Prefer an SSH tunnel (pgAdmin, SSH Tunnel tab) over exposing `5432` to the internet.
- Never commit passwords to this repo.
