
## Project Description

This project features a CI/CD pipeline designed to automate the management of PostgreSQL database backups using the pgBackRest utility. The pipeline performs the following key tasks:

1. Backup Creation: The pipeline automates the process of creating PostgreSQL database backups using pgBackRest, ensuring reliable and efficient backup procedures.

2. Backup Storage: After creation, the backups are automatically uploaded to MinIO object storage. This provides secure and scalable storage for the backup data.

3. Old Backup Cleanup: The pipeline includes a process for automatically cleaning up backups older than 14 days. This helps manage storage space and prevents the accumulation of outdated data.

4. Backup Restoration: The pipeline supports restoring data from backups, with flexibility based on the target server and the specific backup. Point-in-Time Recovery is also supported, allowing data to be restored to a precise moment in time.

Restoration Parameters:
    PG_BACKREST_STANZA: The stanza name for pgBackRest, set manually through the GitLab web interface when initiating the restoration process.
    TARGET_TIME: The target time for Point-in-Time Recovery, set manually through the GitLab web interface when initiating the restoration process.

The pipeline is fully integrated with GitLab CI/CD, providing automation for the creation, storage, and restoration of backups, thereby simplifying the administration and maintenance of PostgreSQL databases.

# VARIABLES:

| Variable              | Value Example            | Description                                 |
|-----------------------|--------------------------|---------------------------------------------|
| `DEMO_HOST`           | `https://demo.example.com`| Host for the demo environment               |
| `DEV_HOST`            | `https://dev.example.com` | Host for the development environment        |
| `KEY`                 | `abc123xyz`               | Primary authentication key                  |
| `KEY2`                | `def456uvw`               | Secondary authentication key                |
| `MINIO_ADMIN_PASSWORD`| `minio-password`          | Password for MinIO admin                    |
| `MINIO_ADMIN_USERNAME`| `minio-admin`             | Username for MinIO admin                    |
| `MINIO_API_URL`       | `https://minio.example.com`| URL for accessing MinIO API                 |
| `PGBACKREST_SERVER`   | `pgbackrest.example.com`  | Server address for pgBackRest               |
| `S3_CONFIG`           | `s3-config.json`          | Configuration file for S3 storage           |
| `S3_CREDS`            | `s3-creds.json`           | Credentials file for S3 storage             |
| `TOKEN`               | `ghp_abc1234567890`       | General access token                        |
| `TRIGGER_TOKEN`       | `trigger-token-xyz`       | Token used to trigger a recovery process    |
| `PG_BACKREST_STANZA`  | `stanza-name`             | Stanza name for pgBackRest, set manually in GitLab UI when starting the recovery process |
| `TARGET_TIME`         | `2024-08-31 12:00:00`     | Target time for point-in-time recovery, set manually in GitLab UI when starting the recovery process |
