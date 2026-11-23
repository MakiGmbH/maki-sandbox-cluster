# Maki Sandbox – Cluster

Die Cluster-Variante erweitert die Basic-Sandbox um einen lokalen Kubernetes-Cluster. **k3d** erstellt den Cluster als Docker-Container. Kubernetes verwaltet darin zwei NGINX-Pods (Replikate). Fällt ein Pod aus, wird er vom Deployment automatisch ersetzt.

Die gesamte Umgebung läuft lokal auf Ihrem Rechner. Sie ist unabhängig von der zentralen Lerninfrastruktur und kann nach dem Herunterladen der benötigten Images auch ohne Internetverbindung weiterbetrieben werden.

## Funktionen

- Lokaler Kubernetes-Cluster mit k3d
- Ein Server-Knoten und ein zusätzlicher Agent-Knoten
- Zwei NGINX-Replikate über ein Kubernetes-Deployment
- `ClusterIP`-Service für die interne Erreichbarkeit der Pods
- Ingress-Regel, die den HTTP-Verkehr über den mitgelieferten Traefik-Ingress-Controller weiterleitet
- Bereitstellung der Konfiguration über Kubernetes-Manifeste in YAML
- Selbstheilung: Kubernetes ersetzt ausgefallene Pods entsprechend dem Deployment

## Voraussetzungen

| Komponente | Mindestanforderung / Hinweis |
|---|---|
| Betriebssystem | macOS (Intel oder Apple Silicon), Windows 11 oder eine gängige 64-Bit-Linux-Distribution |
| Arbeitsspeicher | 8 GB; 16 GB empfohlen |
| Prozessor | 4 CPU-Kerne |
| Speicherplatz | 15 GB frei; 25 GB empfohlen |
| Software | Docker, Git, k3d, kubectl und ein Terminal |
| Rechte | Administrator- bzw. `sudo`-Rechte für die Installation |
| Internet | Für das Klonen des Repositorys und den erstmaligen Download der Container-Images |

Unter Windows wird Docker Desktop mit aktivierter WSL2-Integration empfohlen. Bei Linux muss die Docker Engine laufen und für den angemeldeten Benutzer erreichbar sein. k3d benötigt Docker als Container-Laufzeitumgebung.

## Installation der Voraussetzungen

Installieren Sie Docker und Git sowie zusätzlich k3d und kubectl. Prüfen Sie die Werkzeuge anschließend:

```bash
docker --version
docker compose version
git --version
k3d version
kubectl version --client
```

Weiterführende Anleitungen:

- [Docker Desktop](https://docs.docker.com/desktop/)
- [Docker Engine](https://docs.docker.com/engine/install/)
- [k3d](https://k3d.io/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Git](https://git-scm.com/doc)

### Hinweise zu den Betriebssystemen

- **macOS:** Docker Desktop starten; k3d, kubectl und Git können beispielsweise über Homebrew installiert werden.
- **Windows 11:** WSL2 und Virtualisierung aktivieren, Docker Desktop installieren und die gewünschte WSL-Distribution unter *Settings → Resources → WSL Integration* aktivieren.
- **Linux:** Docker Engine installieren und sicherstellen, dass der eigene Benutzer Docker verwenden darf. Nach einer Änderung der Docker-Gruppenzugehörigkeit ist eine neue Anmeldung erforderlich.

## Sandbox starten

Klonen Sie das Repository und wechseln Sie in das Projektverzeichnis:

```bash
git clone https://github.com/MakiGmbH/maki-sandbox-cluster.git
cd maki-sandbox-cluster
```

Erstellen Sie anschließend den Cluster und wenden Sie die Kubernetes-Manifeste an:

```bash
k3d cluster create maki-sandbox -p "127.0.0.1:8080:80@loadbalancer" --agents 1 --api-port 127.0.0.1:6443
kubectl apply -f kubernetes/
```

Die Portbindung auf `127.0.0.1` beschränkt den Zugriff auf den lokalen Rechner. Wenn Sie den Webserver bewusst im lokalen Netzwerk bereitstellen möchten, können Sie die Host-IP-Bindung entsprechend anpassen.

Der Cluster besteht aus einem Server-Knoten und einem zusätzlichen Agent-Knoten. Der LoadBalancer nimmt HTTP-Anfragen auf Port `8080` entgegen und leitet sie an den Ingress-Controller weiter.

## Betrieb überprüfen

Knotenstatus prüfen:

```bash
kubectl get nodes
```

Pods prüfen:

```bash
kubectl get pods
```

Beide Webserver-Pods sollten nach dem Start den Status `Running` anzeigen. Weitere Ressourcen lassen sich so anzeigen:

```bash
kubectl get deployments,services,ingress
```

Öffnen Sie anschließend im Browser:

**http://localhost:8080**

Bis die Seite verfügbar ist, können je nach Betriebssystem und erstmaligem Image-Download einige Minuten vergehen. Eine vorübergehende `404`-Meldung während des Starts ist möglich.

## Eigene Inhalte und Anwendungen

### Beispielseite ändern

In dieser Variante wird der HTML-Inhalt direkt in der Kubernetes-ConfigMap hinterlegt. Bearbeiten Sie daher:

`kubernetes/webserver-configmap.yaml`

Passen Sie dort den Inhalt unter `data:` und `index.html` an. Wenden Sie die Manifeste erneut an:

```bash
kubectl apply -f kubernetes/
```

**Wichtig:** Die Datei `anwendungen/index.html` im Hauptverzeichnis ist in der aktuellen Cluster-Konfiguration nicht mit der ConfigMap verknüpft. Eine Änderung an dieser Datei allein aktualisiert die laufende Webseite nicht.

### Eigenes Container-Image verwenden

Passen Sie in `kubernetes/deployment.yaml` den Wert `image:` an das gewünschte Image an. Wenn die Anwendung nicht auf Port `80` lauscht, müssen `containerPort` im Deployment sowie `port` und `targetPort` im Service zusammenpassen. Wenden Sie die Änderungen anschließend erneut an:

```bash
kubectl apply -f kubernetes/
```

Die Anzahl der gewünschten Pods wird über `spec.replicas` im Deployment festgelegt.

## Änderungen mit Git sichern

Wenn Sie Ihre Änderungen in einem eigenen GitHub-Repository speichern möchten, legen Sie dort zunächst ein leeres Repository an und ändern Sie die Remote-Adresse:

```bash
git remote set-url origin https://github.com/<BENUTZERNAME>/<REPOSITORY>.git
git status
git add kubernetes/
git commit -m "Kubernetes-Konfiguration angepasst"
git push -u origin main
```

Ersetzen Sie die Platzhalter durch Ihre Angaben. Für HTTPS verwendet GitHub einen Personal Access Token (PAT) statt des Kontopassworts. Speichern Sie den Token niemals in YAML-Dateien, Skripten oder der Remote-URL.

## Sandbox stoppen und entfernen

Der gesamte lokale Kubernetes-Cluster einschließlich seiner Pods und Services wird mit folgendem Befehl entfernt:

```bash
k3d cluster delete maki-sandbox
```

Die Dateien im Repository bleiben erhalten. Zum erneuten Start führen Sie die Befehle aus dem Abschnitt **Sandbox starten** erneut aus.

## Projektstruktur

Die für diese Variante maßgeblichen Kubernetes-Dateien:

```text
maki-sandbox-cluster/
└── kubernetes/
    ├── deployment.yaml
    ├── webserver-configmap.yaml
    ├── webserver-ingress.yaml
    └── webserver-service.yaml
```

Im Repository liegen zusätzlich Dateien aus dem Basic-Grundgerüst (`Dockerfile`, `docker-compose.yml` und `anwendungen/index.html`). Der beschriebene Cluster-Betrieb verwendet stattdessen die Kubernetes-Manifeste; insbesondere stammt der HTML-Inhalt aus `webserver-configmap.yaml`.

## Fehlerbehebung

| Problem | Mögliche Ursache | Lösung |
|---|---|---|
| `Cannot connect to the Docker daemon` | Docker läuft nicht | Docker Desktop bzw. die Docker Engine starten. |
| `k3d cluster create` meldet, dass der Cluster bereits existiert | Ein Cluster gleichen Namens ist vorhanden | Mit `k3d cluster list` prüfen und den bestehenden Cluster verwenden oder mit `k3d cluster delete maki-sandbox` neu erstellen. |
| Kubernetes-API nicht erreichbar / Kubeconfig-Fehler | Cluster wurde neu erstellt oder Konfiguration ist veraltet | Cluster löschen und den Startbefehl mit festem API-Port erneut ausführen. |
| Pods stehen auf `CrashLoopBackOff` | Fehler im Image, Port oder Containerstart | `kubectl describe pod <POD-NAME>` und `kubectl logs <POD-NAME>` prüfen. |
| YAML-Fehler | Falsche Einrückung oder Tabulatoren | Nur Leerzeichen verwenden, üblicherweise zwei pro Ebene. |
| Port `8080` ist nicht erreichbar | Port belegt, Pods noch nicht bereit oder Firewall-Regel | `kubectl get pods`, `kubectl get ingress` und die lokale Portbelegung prüfen. |

## Sicherheit

Die Sandbox ist für Lern- und Testzwecke gedacht, nicht für produktive Dienste oder vertrauliche Daten. Die Kubernetes-API wird durch die Startkonfiguration an `127.0.0.1:6443` gebunden. Der Webserver wird standardmäßig ebenfalls nur lokal veröffentlicht. Speichern Sie keine Zugangsdaten oder Tokens im Repository und prüfen Sie vor jedem Commit mit `git status`, welche Dateien hinzugefügt werden.

## Weitere Dokumentation

- [Kubernetes-Dokumentation](https://kubernetes.io/docs/)
- [k3d-Dokumentation](https://k3d.io/)
- [Docker-Dokumentation](https://docs.docker.com/)
- [Git-Dokumentation](https://git-scm.com/doc)
