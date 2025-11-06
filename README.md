# S3Storage
Object-Storage bei Hetzner mit AWS CLI in der PowerShell einrichten.

AWS CLI unter Windows installieren:<br>
https://awscli.amazonaws.com/AWSCLIV2.msi
```html
aws --version
```
In der Hetzner GUI unter Object-Storage einen Bucket erstellen
Object Lock aktivieren Es verhindert das ändern oder löschen.<p>
Sichtbarkeit Privat<br>

Schutz aktivieren so ist das Bucket vor verrehentlichen löschen in der GUI geschützt.
Zugangsdaten einrichten:
```html
aws configure --profile hetzner
```

AWS Access Key ID: DEIN_AWS_ACCESS_KEY_ID<br>
AWS Secret Access Key: DEIN_AWS_SECRET_ACCESS_KEY<br>
Default region name: eu-central-1<br>
Default output format: json<br>

edit C:\Users\sbin\.aws\config  (UTF‑8 ohne BOM oder ANSI unter Encoding im Notepad++)<br>
[profile hetzner]<br>
region = eu-central-1<br>
output = json<br>
s3 =<br>
    endpoint_url = https://nbg1.your-objectstorage.com

edit C:\Users\sbin\.aws\credentials<br>
[hetzner]<br>
aws_access_key_id = DEIN_KEY<br>
aws_secret_access_key = DEIN_SECRET<br>
```html
aws s3api list-buckets --profile hetzner --endpoint-url https://nbg1.your-objectstorage.com
```
Hetzner S3 Object Lock – Test „TestRetention“<br>
1.) Lokalen Ordner und Testdatei erstellen<br>
```html
New-Item -Path "C:\temp" -ItemType Directory -Force
```
```html
New-Item -Path "C:\temp\TestRetention.txt" -ItemType File -Value "Dies ist TestRetention" -Force
```
2️.) Datei in den Bucket hochladen<br>
```html
aws s3 cp "C:\temp\TestRetention.txt" "s3://bhvhomesupport/TestRetention.txt" --endpoint-url https://nbg1.your-objectstorage.com --profile hetzner
```
3️.) VersionID prüfen<br>
```html
aws s3api list-object-versions --bucket bhvhomesupport --prefix "TestRetention.txt" --endpoint-url https://nbg1.your-objectstorage.com --profile hetzner
```
Ergebniss:<br>
{
    "Versions": [
        {
            "Key": "TestRetention.txt",
            "VersionId": "Vdbt-56vG.-jjyfxggGboS5XaAjsRt5",
            "IsLatest": true
        }
    ]
}<p>
4️-) Retention JSON-Datei erstellen<br>
Erstelle die Datei C:\temp\retention.json mit diesem Inhalt. Die Datei enthält gültiges JSON für Compliance Retention.<br>
{<br>
  "Mode": "COMPLIANCE",<br>
  "RetainUntilDate": "2025-11-06T22:44:43Z"<br>
}<br>
5.) Retention für die Datei setzen<br>
```html
aws s3api put-object-retention --bucket bhvhomesupport --key "TestRetention.txt" --version-id "Vdbt-56vG.-jjyfxggGboS5XaAjsRt5" --retention file://C:\temp\retention.json --endpoint-url https://nbg1.your-objectstorage.com --profile hetzner
```
6.) Retention prüfen<br>
```html
aws s3api get-object-retention --bucket bhvhomesupport --key "TestRetention.txt" --version-id "Vdbt-56vG.-jjyfxggGboS5XaAjsRt5" --endpoint-url https://nbg1.your-objectstorage.com --profile hetzner
```
{<br>
    "Retention": {<br>
        "Mode": "COMPLIANCE",<br>
        "RetainUntilDate": "2025-11-06T22:44:43Z"<br>
    }<br>
}<p>
7.) Test: Datei löschen (soll scheitern)<br>
```html
aws s3api delete-object --bucket bhvhomesupport --key "TestRetention.txt" --version-id "Vdbt-56vG.-jjyfxggGboS5XaAjsRt5" --endpoint-url https://nbg1.your-objectstorage.com --profile hetzner
```
Erwartetes Ergebnis:<br>
An error occurred (AccessDenied) when calling the DeleteObject operation: forbidden by object lock<p>
In der Hetzner lässt die Datei sich aber trotzdem löschen<br>
Warum die Datei im Browser „unsichtbar“ wird<br>
Wenn du versuchst, ein Objekt zu löschen, wird ein Delete Marker erstellt.<br>
Delete Marker = neueste Version, die anzeigt: „Datei gelöscht“.<br>
Alte Version(en) bleiben im Bucket, sind aber nicht die „neueste Version“ → Browser zeigt sie nicht mehr automatisch.<p>
8.) Sichtbar machen ohne neu hochzuladen<br>
Delete Marker entfernen (die alte Version bleibt bestehen)<br>
Du brauchst die VersionID des Delete Markers<br>
```html
aws s3api list-object-versions --bucket bhvhomesupport --prefix "TestRetention.txt" --endpoint-url https://nbg1.your-objectstorage.com --profile hetzner
```
Suche den Eintrag unter "DeleteMarkers"<br>
Notiere die "VersionId" des Delete Markers, z. B. "VersionId": "dM-12345",<br>
Dann entferne den Delete Marker:<br>
```html
aws s3api delete-object --bucket bhvhomesupport --key "TestRetention.txt" --version-id "dM-12345" --endpoint-url https://nbg1.your-objectstorage.com --profile hetzner
```
"DeleteMarker": "true" <br>
"VersionId": "dM-12345" <br>
Alte Version wird wieder die neueste Version<br>
Datei wird im Browser wieder angezeigt<br>
Object Lock / Retention bleibt aktiv<br><p>

(neu hochladen – optional Datei wieder sichtbar machen)<br>
```html
aws s3 cp "C:\temp\TestRetention.txt" "s3://bhvhomesupport/TestRetention.txt" --endpoint-url https://nbg1.your-objectstorage.com --profile hetzner
```
Neue Version hat noch keine Retention, Datei wieder sichtbar im Browser. Alte Version bleibt geschützt.<p>

Zusammenfassung der Befehle:<p>

1	Lokalen Ordner & Datei erstellen	New-Item ...	Datei TestRetention.txt existiert<br>
2	Datei hochladen	aws s3 cp ...	Datei im Bucket vorhanden<br>
3	Version prüfen	aws s3api list-object-versions ...	VersionId notieren<br>
4	Retention JSON erstellen	Inhalt in C:\temp\retention.json	Gültiges JSON<br>
5	Retention setzen	aws s3api put-object-retention ...	Retention aktiv<br>
6	Retention prüfen	aws s3api get-object-retention ...	Compliance Mode angezeigt<br>
7	Datei löschen testen	aws s3api delete-object ...	AccessDenied, Datei bleibt geschützt<br>
8 Delete Marker entfernen	aws s3api delete-object ...	Datei wieder sichtbar, Retention bleibt<br>










