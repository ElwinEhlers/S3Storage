# Backup‑Setup für Zammad mit restic auf Hetzner S3

## Installation  
```html
apt update  
apt install restic  
```

Umgebungsvariablen setzen
```html
export AWS_ACCESS_KEY_ID="QWSY7SMWCT27DGIWYLXE"  
export AWS_SECRET_ACCESS_KEY="sB7Xrwdid5TcSZLOxS3bfj9qJsNuutR1z8Dj73oL"  
export RESTIC_PASSWORD="Sichern1"  
export RESTIC_REPOSITORY="s3:bhvhomesupport:/zammad-backup"  
export RESTIC_S3_ENDPOINT="https://nbg1.your-objectstorage.com"  
```

Repository initialisieren
```html
restic init  
```
Datenbank dumpen

```html
sudo -u postgres pg_dump zammad > /var/backups/zammad-db.sql  
```

Backup starten
```html
restic -r $RESTIC_REPOSITORY backup /opt/zammad /var/backups/zammad-db.sql  
```
Snapshots anzeigen
```html
restic -r $RESTIC_REPOSITORY snapshots  
```
Zurückspielen:
```html
restic -r $RESTIC_REPOSITORY restore latest --target /opt/zammad
```
Nur die db
```html
restic -r $RESTIC_REPOSITORY restore latest --target /var/backups --include /var/backups/zammad-db.sql
```
