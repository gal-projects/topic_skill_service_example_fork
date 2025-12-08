# 📘 `README-k8s.md`

## Kubernetes-Konfigurationsvorlagen für Topic & Skill Service (GKE Deployment)

Dieses Projekt verwendet **Kubernetes Templates** (`*.yaml.tpl` Dateien), die über GitHub Actions automatisch mit deinen eigenen Werten befüllt und in **Google Kubernetes Engine (GKE)** deployed werden.

Du musst **NICHT** direkt mit `kubectl` arbeiten.
Die CI/CD-Pipeline übernimmt alles für dich.

---

# 🧭 Wie funktioniert das?

1. Du pflegst die Kubernetes-Templates im Ordner **`k8s/`**.
2. GitHub Actions ersetzt die Platzhalter:

   * `${K8S_NAMESPACE}` → dein Namespace (z. B. `topic-skill-deinname`)
   * `${IMAGE}` → dein Docker-Image aus Docker Hub
3. GitHub Actions deployt die fertigen YAMLs nach GKE:

   * Postgres-Datenbank
   * Topic & Skill Service API
   * LoadBalancer-Service (damit du eine **öffentliche IP** bekommst)

Nach dem Deployment zeigt die Pipeline dir automatisch an, **unter welcher IP deine API erreichbar ist**.

---

# 🔧 Platzhalter & Variablen

Diese Templates enthalten **Platzhalter**, die NICHT direkt ersetzt werden dürfen:

| Platzhalter        | Bedeutung                                         |
| ------------------ | ------------------------------------------------- |
| `${K8S_NAMESPACE}` | Dein persönlicher Namespace im Kubernetes-Cluster |
| `${IMAGE}`         | Dein Docker-Image, das die Pipeline gebaut hat    |

GitHub Actions setzt diese Werte automatisch ein.

---

# 📂 Dateien im Ordner `k8s/` und ihre Aufgabe

Hier findest du eine Übersicht über alle Templates, damit du verstehst, was deployed wird.

---

## 1️⃣ `namespace.yaml.tpl`

Erstellt deinen **Namespace**, in dem alle Ressourcen laufen.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ${K8S_NAMESPACE}
```

Jede:r Student:in erhält einen eigenen Namespace → keinerlei Konflikte.

---

## 2️⃣ `postgres-secret.yaml.tpl`

Speichert die Zugangsdaten für Postgres als Kubernetes-Secret.

```yaml
POSTGRES_USER: app
POSTGRES_PASSWORD: app123
POSTGRES_DB: topics_db
```

Wird vom Postgres-Container automatisch genutzt.

---

## 3️⃣ `app-db-url.yaml.tpl`

Dieses Secret enthält die **Connection URL**, die die Flask-App benötigt:

```text
postgresql+psycopg2://app:app123@postgres:5432/topics_db
```

Der Hostname `postgres` entspricht dem Service-Namen von Postgres.

---

## 4️⃣ `postgres.yaml.tpl`

Startet die **Postgres-Datenbank**:

* `StatefulSet` mit persistentem Storage
* `Service` für internen Zugriff
* Health Checks
* Der Hostname im Cluster lautet: `postgres`

Die App verbindet sich automatisch mit dieser Datenbank.

---

## 5️⃣ `deployment.yaml.tpl`

Das Deployment für die Flask-App:

* Container-Image wird über `${IMAGE}` gesetzt
* Environment Variable `DATABASE_URL` kommt aus dem Secret
* Healthz-Endpoint (`/healthz`) wird überwacht
* 2 Replikas für Stabilität
* Standard-Port: **5000**

Dies ist der Haupt-Service deines Projektes.

---

## 6️⃣ `service.yaml.tpl`

Der öffentliche Service, der eine **LoadBalancer-IP** von GKE erhält.

```yaml
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 5000
```

Damit erhältst du eine **öffentliche URL**, z. B.:

```
http://34.159.xxx.xxx/
http://34.159.xxx.xxx/topics
http://34.159.xxx.xxx/skills
```

Diese IP zeigt dir der Workflow automatisch am Ende an.

---

# 🚀 Was passiert beim Deployment?

Die GitHub Actions Pipeline führt:

1. **Docker Build & Push**
2. **Authentifizierung bei Google Cloud**
3. **`envsubst` → Platzhalter ersetzen**
4. **`kubectl apply -f -` → Deployment**
5. **Warten auf LoadBalancer-IP**
6. **Ausgabe deiner API-URL**

Du musst nur:

* dein Docker Hub Token setzen
* deinen Namespace definieren
* Änderungen committen und pushen

Alles andere erledigt die Pipeline.

---

# 🧪 Wie greife ich auf meine API zu?

Nachdem der Workflow abgeschlossen ist, findest du im GitHub Actions Log:

```
🎉 Dein Service wurde erfolgreich deployed!
Extern erreichbare URL:
http://<EXTERNAL-IP>/
```

Testbare Endpunkte:

| Zweck          | URL        |
| -------------- | ---------- |
| Healthcheck    | `/healthz` |
| Topics abrufen | `/topics`  |
| Skills abrufen | `/skills`  |

Beispiel:

```
http://34.159.85.123/topics
```

---

# ❓ Probleme oder Fragen?

Häufige Fehler:

* **Service hat keine externe IP:**
  Warte 1–2 Minuten, GKE braucht manchmal etwas länger.
* **Namespace existiert nicht:**
  Der Workflow legt ihn automatisch an.
* **Image nicht gefunden:**
  Prüfe dein Docker Hub Repository und Secrets (`DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`).

---


