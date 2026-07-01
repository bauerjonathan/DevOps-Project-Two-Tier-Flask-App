# DevOps-Projekt: Two-Tier Flask App CI/CD – Migration von AWS auf Azure

## 1. Was sich ändert (Mapping AWS → Azure)

| AWS-Komponente | Azure-Äquivalent |
|---|---|
| EC2 Instance (t2.micro, Ubuntu 22.04 AMI) | Azure VM (Standard_B1s, Ubuntu 22.04 LTS Image) |
| Security Group | Network Security Group (NSG) |
| Key Pair (.pem) | SSH-Public-Key (Azure erzeugt .pem-analoges Format) |
| Elastic/Public IP | Azure Public IP (Standard SKU) |
| EC2 Console | Azure Portal / Azure CLI |

Docker, Docker Compose, Jenkins, der Flask-Code, das `Dockerfile`, `docker-compose.yml` und das `Jenkinsfile` bleiben **inhaltlich fast unverändert** – nur die Infrastruktur darunter ist anders.

---

## 2. Architekturdiagramm (Azure-Version)

```
+-----------------+      +----------------------+      +-----------------------------+
|   Developer     |----->|     GitHub Repo      |----->|        Jenkins Server       |
| (pushes code)   |      | (Source Code Mgmt)   |      |  (auf Azure VM)             |
+-----------------+      +----------------------+      |                             |
                                                       | 1. Clont Repo                |
                                                       | 2. Baut Docker Image         |
                                                       | 3. Führt Docker Compose aus  |
                                                       +--------------+--------------+
                                                                      |
                                                                      | Deployt
                                                                      v
                                                       +-----------------------------+
                                                       |   Application Server        |
                                                       |   (dieselbe Azure VM)       |
                                                       |                             |
                                                       | +-------------------------+ |
                                                       | | Container: Flask        | |
                                                       | +-------------------------+ |
                                                       |              |              |
                                                       |              v              |
                                                       | +-------------------------+ |
                                                       | | Container: MySQL        | |
                                                       | +-------------------------+ |
                                                       +-----------------------------+
```

---

## 3. Voraussetzungen

- Azure-Konto (Free Tier reicht: B1s-VM ist 750h/Monat 12 Monate kostenlos)
- Installiert: [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli) (`az`) – oder du nutzt nur das Portal
- Ein SSH-Schlüsselpaar lokal (`ssh-keygen -t rsa -b 4096`), falls noch nicht vorhanden
- GitHub-Repo mit deinem Code (Fork des Original-Repos oder eigenes)

---

## 4. Schritt 1: Ressourcengruppe und VM erstellen

### Option A: Azure CLI (empfohlen, reproduzierbar)

```bash
# Login
az login

# Ressourcengruppe anlegen
az group create --name rg-flask-devops --location germanywestcentral

# VM erstellen (Ubuntu 22.04, B1s = Free-Tier-fähig)
az vm create \
  --resource-group rg-flask-devops \
  --name vm-flask-jenkins \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username azureuser \
  --ssh-key-values ~/.ssh/id_rsa.pub \
  --public-ip-sku Standard
```

Der Befehl gibt dir am Ende die `publicIpAddress` zurück – die brauchst du gleich.

### Option B: Azure Portal

1. Portal → "Virtuelle Computer erstellen"
2. **Ressourcengruppe**: neu anlegen, z. B. `rg-flask-devops`
3. **VM-Name**: `vm-flask-jenkins`
4. **Region**: Germany West Central (o. ä., nah an dir)
5. **Image**: Ubuntu Server 22.04 LTS – x64 Gen2
6. **Größe**: Standard_B1s (Free-Tier-berechtigt) – für mehr Puffer beim Jenkins-Build ggf. B2s
7. **Authentifizierungstyp**: SSH public key, deinen lokalen Public Key einfügen oder neuen generieren lassen
8. **Inbound-Portregeln**: hier NICHTS vorkonfigurieren, das machen wir gezielt in Schritt 5 über die NSG
9. Erstellen klicken, warten bis Deployment fertig ist, Public IP notieren

---

## 5. Schritt 2: Netzwerksicherheitsgruppe (NSG) konfigurieren

Das ist das Pendant zur AWS Security Group. Du brauchst 4 eingehende Regeln:

| Priorität | Name | Port | Protokoll | Quelle |
|---|---|---|---|---|
| 100 | Allow-SSH | 22 | TCP | Deine IP (nicht "Any"!) |
| 110 | Allow-HTTP | 80 | TCP | Any |
| 120 | Allow-Flask | 5000 | TCP | Any |
| 130 | Allow-Jenkins | 8080 | TCP | Any |

### Per CLI:

```bash
az network nsg rule create --resource-group rg-flask-devops \
  --nsg-name vm-flask-jenkinsNSG --name Allow-SSH \
  --priority 100 --destination-port-ranges 22 --protocol Tcp \
  --source-address-prefixes "<DEINE-IP>/32" --access Allow

az network nsg rule create --resource-group rg-flask-devops \
  --nsg-name vm-flask-jenkinsNSG --name Allow-HTTP \
  --priority 110 --destination-port-ranges 80 --protocol Tcp \
  --source-address-prefixes "*" --access Allow

az network nsg rule create --resource-group rg-flask-devops \
  --nsg-name vm-flask-jenkinsNSG --name Allow-Flask \
  --priority 120 --destination-port-ranges 5000 --protocol Tcp \
  --source-address-prefixes "*" --access Allow

az network nsg rule create --resource-group rg-flask-devops \
  --nsg-name vm-flask-jenkinsNSG --name Allow-Jenkins \
  --priority 130 --destination-port-ranges 8080 --protocol Tcp \
  --source-address-prefixes "*" --access Allow
```

> **Sicherheitshinweis**: Bei AWS wird im Original-Tutorial SSH auf "Anywhere" freigegeben – das würde ich für Azure nicht übernehmen. Schränk Port 22 auf deine eigene IP ein (`curl ifconfig.me` zeigt sie dir).

### Per Portal:
VM → "Netzwerk" → "Netzwerkeinstellungen hinzufügen" → die vier Regeln wie in der Tabelle anlegen.

---

## 6. Schritt 3: Mit der VM verbinden

```bash
ssh azureuser@<PUBLIC-IP>
```

Statt `ubuntu@<ec2-ip>` (AWS) also `azureuser@<azure-ip>` (oder der Username, den du bei der VM-Erstellung gewählt hast).

---

## 7. Schritt 4: Abhängigkeiten installieren (identisch zu AWS)

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install git docker.io docker-compose-v2 -y
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker $USER
newgrp docker
```

---

## 8. Schritt 5: Jenkins installieren (identisch zu AWS)

```bash
sudo apt install openjdk-17-jdk -y

curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update
sudo apt install jenkins -y
sudo systemctl start jenkins
sudo systemctl enable jenkins

# Initiales Passwort holen
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Dashboard aufrufen: `http://<PUBLIC-IP>:8080`, Passwort einfügen, "Install suggested plugins", Admin-User anlegen.

```bash
# Jenkins Docker-Rechte geben
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

---

## 9. Schritt 6: GitHub-Repo-Dateien (keine inhaltliche Änderung nötig)

Die drei Dateien aus dem Original funktionieren unverändert unter Azure, da sie keine AWS-spezifischen Services nutzen:

**Dockerfile** – bleibt exakt gleich.

**docker-compose.yml** – bleibt exakt gleich.

**Jenkinsfile** – nur die Repo-URL anpassen:

```groovy
pipeline {
    agent any
    stages {
        stage('Clone Code') {
            steps {
                git branch: 'main', url: 'https://github.com/<dein-username>/<dein-repo>.git'
            }
        }
        stage('Build Docker Image') {
            steps {
                sh 'docker build -t flask-app:latest .'
            }
        }
        stage('Deploy with Docker Compose') {
            steps {
                sh 'docker compose down || true'
                sh 'docker compose up -d --build'
            }
        }
    }
}
```

---

## 10. Schritt 7: Jenkins-Pipeline erstellen

1. Jenkins-Dashboard → "New Item"
2. Name vergeben, Typ **Pipeline** wählen, OK
3. Im Pipeline-Bereich: **Definition** = "Pipeline script from SCM"
4. **SCM** = Git
5. Repository-URL eintragen
6. **Script Path** = `Jenkinsfile` (Standard, meist schon korrekt)
7. Speichern

Optional, aber empfehlenswert für "echtes" CI/CD: **GitHub-Webhook** einrichten, damit Jenkins bei jedem Push automatisch baut, statt nur manuell per "Build Now":
- GitHub-Repo → Settings → Webhooks → Add webhook
- Payload-URL: `http://<PUBLIC-IP>:8080/github-webhook/`
- Content type: `application/json`
- Im Jenkins-Job unter "Build Triggers": "GitHub hook trigger for GITScm polling" aktivieren

---

## 11. Schritt 8: Pipeline ausführen & Deployment prüfen

- "Build Now" klicken, Stage View / Console Output beobachten
- Nach erfolgreichem Build: `http://<PUBLIC-IP>:5000`
- Auf der VM: `docker ps` zeigt beide laufenden Container (Flask + MySQL)

---

## 12. Azure-spezifische Erweiterungen (optional, aber sinnvoll für dein Portfolio)

Da du das ohnehin als Lern-/Portfolio-Projekt nutzt, ein paar Punkte, die das Ganze "Azure-nativer" statt nur "AWS-Tutorial-mit-anderem-Provider" machen:

- **Azure Container Registry (ACR)** statt lokalem Docker-Build: Images in ACR pushen, Jenkins zieht von dort – näher an echten Enterprise-Setups.
- **Azure Monitor / Log Analytics**: VM-Diagnostics aktivieren, um Metriken/Logs zentral zu sehen statt nur `docker logs`.
- **Managed Disk Snapshots** statt manueller Backups der MySQL-Daten.
- **Azure DevOps Pipelines** als Alternative zu Jenkins – spart dir die VM-Wartung für den CI-Server komplett (guter Vergleichspunkt für Bewerbungsgespräche: "warum Jenkins vs. Azure DevOps").
- **NSG Flow Logs** statt Security-Group-Standardlogging, wenn du Netzwerktraffic analysieren willst.

## 13. Kosten im Blick behalten

- B1s-VM: im Free Tier 750h/Monat für 12 Monate kostenlos, danach ca. 0,008 €/h
- Public IP (Standard SKU): läuft auch im Leerlauf Kosten, wenn nicht im Free-Tier-Kontingent
- Nach dem Testen: `az group delete --name rg-flask-devops --yes --no-wait` löscht alle Ressourcen der Gruppe auf einmal – guter Reflex, um keine Kosten liegen zu lassen
