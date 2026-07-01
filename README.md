# Two-Tier Flask App – Automatisierte CI/CD-Pipeline auf Microsoft Azure

## 📋 Projektbeschreibung

Dieses Projekt zeigt den vollständigen Aufbau einer automatisierten CI/CD-Pipeline für eine containerisierte Two-Tier-Webanwendung (Flask + MySQL) auf **Microsoft Azure**. Die Anwendung läuft in Docker-Containern, wird über Docker Compose orchestriert und bei jedem Push auf den `main`-Branch automatisch über Jenkins gebaut und deployt.

Ziel des Projekts war es, ein realistisches DevOps-Setup end-to-end selbst umzusetzen: von der Infrastruktur-Provisionierung über Container-Orchestrierung bis zur vollautomatisierten Pipeline mit GitHub-Webhook-Integration.

---

## 🏗️ Architektur

```
Developer (Push)
      │
      ▼
GitHub Repository
      │  Webhook-Trigger
      ▼
Jenkins (auf Azure VM)
      │  1. Clont Repository
      │  2. Baut Docker-Image
      │  3. Führt Docker Compose aus
      ▼
Application Server (dieselbe Azure VM)
      ├── Container: Flask (Port 5000)
      └── Container: MySQL (Port 3306, persistentes Volume)
```

---

## 🛠️ Tech Stack

| Bereich | Technologie |
|---|---|
| Cloud-Plattform | Microsoft Azure (Virtual Machine, Ubuntu 22.04 LTS) |
| Backend | Python / Flask |
| Datenbank | MySQL |
| Containerisierung | Docker, Docker Compose |
| CI/CD | Jenkins (Pipeline as Code, GitHub-Webhook) |
| Versionierung | Git / GitHub |
| Netzwerksicherheit | Azure Network Security Group (NSG) |

---

## ⚙️ Infrastruktur-Setup

Die Anwendung läuft auf einer **Azure Virtual Machine** (Standard_B1s, Ubuntu 22.04 LTS). Der Zugriff erfolgt über gezielt konfigurierte NSG-Regeln:

| Port | Zweck | Quelle |
|---|---|---|
| 22 | SSH-Zugriff | eingeschränkt auf eigene IP |
| 80 | HTTP | offen |
| 5000 | Flask-Anwendung | offen |
| 8080 | Jenkins-Dashboard | offen |

Auf der VM laufen Docker, Docker Compose und Jenkins direkt auf dem Host-System; die Anwendung selbst läuft vollständig containerisiert.

---

## 🚀 CI/CD-Pipeline

Die Pipeline ist als **Jenkinsfile** (Pipeline as Code) im Repository definiert und durchläuft folgende Stages:

1. **Clone Code** – Repository wird von GitHub geklont
2. **Build Docker Image** – Flask-Image wird gebaut
3. **Deploy with Docker Compose** – bestehende Container werden gestoppt, neue Version wird hochgefahren

Ausgelöst wird die Pipeline automatisch über einen **GitHub-Webhook**: jeder Push auf `main` löst sofort einen neuen Build und ein automatisches Redeployment aus – kein manuelles Eingreifen nötig.

---

## 💾 Datenpersistenz

MySQL-Daten überleben Container-Neustarts durch ein Docker-Volume (`mysql-data:/var/lib/mysql`). Das Docker-Volume liegt unabhängig vom Container-Lebenszyklus auf der VM – ein `docker compose down` oder Container-Update löscht die gespeicherten Daten also nicht.

---

## 📦 Lokales Setup

```bash
# Repository klonen
git clone https://github.com/bauerjonathan/DevOps-Project-Two-Tier-Flask-App.git
cd DevOps-Project-Two-Tier-Flask-App

# Container bauen und starten
docker compose up -d --build

# Anwendung ist erreichbar unter:
# http://localhost:5000
```

---

## 🔧 Deployment auf Azure (Kurzfassung)

1. Azure VM erstellen (Ubuntu 22.04, Standard_B1s)
2. NSG-Regeln für Ports 22, 80, 5000, 8080 konfigurieren
3. Docker, Docker Compose und Jenkins auf der VM installieren
4. GitHub-Repository mit Jenkins-Pipeline verknüpfen
5. GitHub-Webhook für automatische Builds einrichten
6. Pipeline testen: Push auf `main` → automatischer Build & Deploy

---

## 📌 Learnings aus dem Projekt

- Praktische Erfahrung mit Infrastruktur-Provisionierung auf Azure (VM, NSG, Public/Private IP)
- Umgang mit Container-Orchestrierung via Docker Compose (Networking, Volumes, Health Checks)
- Aufbau einer vollautomatisierten CI/CD-Pipeline mit Jenkins inkl. Webhook-Integration
- Troubleshooting typischer Real-World-Probleme: GPG-Key-Rotation bei Paketquellen, Java-Versionskompatibilität, Branch-Mismatches in der Pipeline-Konfiguration

---

## 👤 Autor

**Jonathan Bauer**
[GitHub](https://github.com/bauerjonathan) · [LinkedIn](https://linkedin.com/in/jonathan-bauer-7b2752369)