# WPMII - Cloud Computing

Ziel der Lehrveranstaltung ist es, die Amazon AWS Fähigkeiten aus dem letzten Semester zu vertiefen.  
Dazu soll zum einen der aufbauende Lehrgang "AWS Academy Cloud Developing" in der "AWS Academy" abgeschlossen werden (kann Note um 0,5 Punke verbessern), zum anderen soll eine Aufgabe automatisiert in der Cloud ausgeführt werden.

- [WPMII - Cloud Computing](#wpmii---cloud-computing)
  - [AWS Skript](#aws-skript)
    - [Aufgabe](#aufgabe)
    - [Möglicher Lösungsansatz](#möglicher-lösungsansatz)
    - [AWS-CLI Befehle](#aws-cli-befehle)
  - [AWS Academy Lehrgang](#aws-academy-lehrgang)
    - [Wissenschecks](#wissenschecks)
    - [Labore](#labore)

## AWS Skript

### Aufgabe

Ich als Service-Techniker  
möchte ein Simulations-Programm konfigurieren, einen Rechner starten, das Programm ausführen und die Ergebnisse herunterladen,  
um bei gegebener Komplexität genug "Power" zu haben, ohne einen HW-Invest zu tätigen.  
Ich möchte diesen eignen Service in unregelmäßigen Abständen ohne Zutun eines AWS-Experten verwenden können.

### Möglicher Lösungsansatz

- Amazon CLI mit PowerShell-Skript unter Windows steuern
- Notwendige Daten am Anfang ermitteln und prüfen
- Sicherstellen, dass SSH-Schlüssel zum Verbinden vorhanden ist
- AWS Netzwerk konfigurieren
  - Öffentliches Subnet ermitteln
  - Sicherheitsgruppe
  - Firewall Regeln
- EC2 Instanz starten
  - Deren IP ermitteln
- Aufgabe per SSH auf den Server hochladen
- Aufgabe per SSH auf dem Server starten
- Ergebnis per SSH herunterladen und Sitzung schließen
- EC2 Instanz wieder stoppen (damit keine weiteren Kosten entstehen)
- Regeln aus Security Group und anschließend diese selbst löschen

### AWS-CLI Befehle

*Befehle enthalten PowerShell-Variablen ($...) - diese müssen auf die benutze Sprache angepasst werden*  
Durch die Befehle wird (hier von der PowerShell) die lokal installierte AWS-CLI aufgerufen.

`aws ec2 create-key-pair --key-name $awsuser --key-type rsa --key-format pem --query "KeyMaterial" --output text > $key`  
Erstellt einen SSH-Schlüssel in der Datei, deren Pfad in `$key` steht.

```PowerShell
$subnets = (aws ec2 describe-subnets | ConvertFrom-Json) # alle subnets abfragen
$subnetid = ""
$i = 0
while ($i -lt $subnets.Subnets.Count) # über subnets schleifen
{
    if ($subnets.Subnets[$i].AvailabilityZone -like "$awsregion*" -and $subnets.Subnets[$i].MapPublicIpOnLaunch -eq "true") { # das erste wählen, das in der passenden region ist und öffentliche ips vergibt
        $subnetid = $subnets.Subnets[$i].SubnetId
        break
    }
    $i = $i + 1
}
if ($subnetid -eq "") {
    Write-Error "Es konnte kein öffentliches Subnetz bezogen werden! Bitte kontaktieren Sie Ihren AWS-Administrator."
    exit
}
```

Fragt die verfügbaren Subnets des Accounts ab und nutzt das Erste. Instanz wird diesem zugewiesen.

`$secgroup = aws ec2 create-security-group --group-name "securitygroup-$awsuser" --description "security group of $awsuser"`  
Erstellt eine neue Security Group

`aws ec2 authorize-security-group-ingress --group-id ($secgroup | ConvertFrom-Json).GroupId --protocol tcp --port 22 --cidr 0.0.0.0/0`  
Weist der Security Group die Regel zu, dass jede IPv4 über den Port 22 auf die Instanz zugreifen darf.

`$instance = aws ec2 run-instances --image-id $awsami --count 1 --instance-type $awsinstancetype --key-name $awsuser --security-group-ids ($secgroup | ConvertFrom-Json).GroupId --subnet-id $subnetid`  
Startet eine neue EC2-Instanz. Als Image ist "Amazon Linux 2" zu empfehlen - ACHTUNG: AMI-IDs sind regionsspezifisch.

`aws ec2 create-tags --resources ($instance | ConvertFrom-Json).Instances[0].InstanceId  --tags Key=DHGE,Value=$awsuser`  
Der Dozent wollte die Instanzen getaggt haben. Sonst keine funktionale Auswirkung.

`$pubip = (aws ec2 describe-instances --filter "Name=instance-id,Values=$(($instance | ConvertFrom-Json).Instances[0].InstanceId)" --query "Reservations[*].Instances[*].PublicIpAddress | ConvertFrom-Json)[0][0]"`  
Ermittelt die öffentliche IP der Instanz. Wird zum Verbinden benötigt.

```PowerShell
$nopasswd = New-Object System.Security.SecureString # leeres password initalisieren
$credential= New-Object System.Management.Automation.PSCredential ($awsamiuser, $nopasswd) # logindaten vorbereiten
$session = New-SSHSession –ComputerName $pubip -KeyFile $key -Credential $credential -ConnectionTimeout 10 -Force # https://github.com/darkoperator/Posh-SSH/blob/master/docs/New-SSHSession.md

Set-SCPItem –ComputerName $pubip -KeyFile $key -Credential $credential -Force -Path "$PSScriptRoot\task.zip" -Destination '~'
```

SSH Verbindung zum Server herstellen und Archiv mit Aufgabe hochladen.

```PowerShell
Invoke-SSHCommand -Command "unzip ~/task.zip -d ~/task/" -SSHSession $session
Invoke-SSHCommand -Command "sh ~/task/task.sh" -SSHSession $session
```

Aufgabe entpacken und ausführen.

```PowerShell
Get-SCPItem –ComputerName $pubip -KeyFile $key -Credential $credential -Force -Path "~/task/result.json" -PathType File -Destination "$PSScriptRoot" # https://github.com/darkoperator/Posh-SSH/blob/master/docs/Get-SCPItem.md
Remove-SSHSession -SSHSession $session
```

Ergebnisdatei vom Server herunterladen und Sitzung freigeben.

```PowerShell
aws ec2 terminate-instances --instance-ids ($instance | ConvertFrom-Json).Instances[0].InstanceId
aws ec2 revoke-security-group-ingress --group-id ($secgroup | ConvertFrom-Json).GroupId --protocol tcp --port 22 --cidr 0.0.0.0/0
Start-Sleep -Seconds 60
aws ec2 delete-security-group --group-id ($secgroup | ConvertFrom-Json).GroupId
```

Instanz terminieren (quasi löschen, wird nach einiger Zeit automatisch aus Webkonsole gelöscht).  Anschließend Regeln von Security Group löschen. Dann warten, bis EC2-Instanz heruntergfahren (auf gut Glück 60 Sekunden warten), und diese löschen.

## AWS Academy Lehrgang

Folgender AWS Kurs soll bearbeitet werden. Dies kann die Note um bis zu 0,5 Punkte verbessern.

[AWS Academy Cloud Developing [25947]](https://awsacademy.instructure.com/courses/25947)

### Wissenschecks

1. Modul 2
   1. 4 (Develop, deploy, maintaun)
   2. 4 (New components are ...)
   3. 3 (IAM)
   4. (AWS Cloud9)
   5. 3,4,6
   6. 2,3,6
   7. 1 (True)
   8. ?
   9. 1
   10. 4 (400)
2. Modul 3
   1. 2 (Data lake)
   2. 2 (Bucket)
   3. 4
   4. 4
   5. 2 (False)
   6. 1 (5GB)
   7. 3 (Bucket policy)
   8. ?
   9. 3 (SSL/TLS)
   10. 4 (CORS)
3. Modul 4
   1. 1,3
   2. 4 (IAM policies)
   3. 2
   4. 1,2
   5. 3
   6. 4
   7. 1
   8. 3
   9. 1
   10. 3,4
4. Modul 5
   1. 4
   2. 3
   3. 4
   4. 2
   5. 2
   6. 4
   7. 3,4
   8. 3
   9. 3
   10. 1
5. Modul 6
   1. 3
   2. 3
   3. 1
   4. 2
   5. 2
   6. 1
   7. 1
   8. 13
   9. 4
   10. 3
6. Modul 7
   1. 1
   2. 3
   3. 1
   4. ?
   5. 1
   6. 2
   7. 3,5
   8. 3
   9. 2
   10. 4
7. Modul 8
   1. 2
   2. 2,4
   3. 1
   4. 1
   5. 2
   6. 1
   7. 1
   8. 1
   9. 3
   10. 3
8. Modul 9
   1. 3
   2. 3
   3. 2
   4. 3
   5. 2
   6. 2
   7. 1
   8. 2
   9. 1
   10. 1
9. Modul 10
   1. 2
   2. 4
   3. 2
   4. 1
   5. 2
   6. 4
   7. 1
   8. 3
   9. 2
   10. 2
10. Modul 11
    1. 2
    2. 4
    3. 2
    4. 3
    5. 3
    6. 1
    7. 1
    8. 3
    9. 4
    10. 1
11. Modul 12
    1. 3
    2. 1
    3. 2
    4. 2
    5. 2
    6. 1
    7. 2
    8. 4
    9. 1
    10. 2
12. Modul 13
    1. 2
    2. 1
    3. 4
    4. 2
    5. 3
    6. 4
    7. 2
    8. 1
    9. 2
    10. 2

### Labore

Einfach der Beschreibung in der Academy folgen und am Ende auf Submit klicken.  
**Achtung**: Nicht aus Versehen bei Benennungen was verdrehen, sonst darf man alles noch mal machen.
