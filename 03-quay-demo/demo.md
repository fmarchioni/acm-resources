# Lab Quay Standalone

## Obiettivo

Lo scopo di questo laboratorio è di utilizzare un Server Quay per eseguire operazioni di push/pull immagini utilizzando sia podman che le API REST di Quay
Questi aspetti sono gestiti con l'utenza già disponibile nel laboratorio (developer) e creando altre utenze/teams/organizations 

Riferimenti: il server Quay è disponibile su: **https://registry.ocp4.example.com:8443**

**Pre-requisito:** prova ad accedere dal browser con l’utente `developer/developer`.

---

## 1) Login da Podman

```bash
podman login -u developer -p developer https://registry.ocp4.example.com:8443
````

---

## 2) Pull immagine di test (Alpine)

```bash
podman pull quay.io/libpod/alpine:latest
```

---

## 3) Tag dell’immagine per il tuo registry

```bash
podman tag alpine:latest registry.ocp4.example.com:8443/developer/alpine:latest
```

---

## 4) Push dell’immagine

```bash
podman push --tls-verify=false registry.ocp4.example.com:8443/developer/alpine:latest
```

---

## 5) Preparazione alle API REST

Per usare le API REST di Quay, serve un **token OAuth**.
I passaggi principali:

1. Crea un **Robot User** `ocp-robot` dalla GUI con l'utente **developer**
2. Salva il token su disco
3. Login di prova con token:

```bash
QUAY_HOST="registry.ocp4.example.com:8443"
REPO="developer/alpine"
ROBOT_USER="developer+ocprobot"
ROBOT_PASS="AH0NZ82DJUDYC3WH03DKEPAG2I4Y87N5HN0L6X47LLY45E2QEMSTLD2Q7V7WA0C4" # notsecret


TOKEN=$(curl -s -k -u "$ROBOT_USER:$ROBOT_PASS" \
  "https://$QUAY_HOST/v2/auth?service=$QUAY_HOST&scope=repository:$REPO:pull" | jq -r .token)
```


---

## 7) Test token: elenco API disponibili

```bash
curl -k -s -H "Authorization: Bearer $TOKEN" \
 "https://$QUAY_HOST/api/v1/discovery" | jq .
```

---

## 8) Esempio uso API Quay: scaricare il manifest dell’immagine Alpine

```bash
curl -k -s -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/vnd.docker.distribution.manifest.v2+json" \
  "https://$QUAY_HOST/v2/$REPO/manifests/latest" | jq .
```

---

## 9) Esempio uso API Quay: Vedere i tags disponibili

```bash
curl -s -k -H "Authorization: Bearer $TOKEN" \
  "https://$QUAY_HOST/v2/$REPO/tags/list" | jq .
```

# Parte II: Creazione utenze / Organization / Teams

Questa sezione mostra l’utilizzo di Red Hat Quay per:

- creare un’Organization  
- creare un Team  
- aggiungere utenti  
- assegnare permessi  
- pushare un’immagine in un repository dell’Organization  

L’obiettivo è comprendere come Quay gestisce la collaborazione tra utenti tramite Team e permessi.

## 1) Creazione nuovo utente admin (GUI)

* Username: `admin`
* Password: `redhat123`


Questo utente verrà utilizzato per la gestione dell’Organization.

---

## 2) Creazione di una Organization
Accedere alla GUI come **admin**.

Dal menu principale selezionare **Create Organization** e inserire:

- **Name:** `finance`  
- **Email:** `finance@example.com`

La nuova Organization apparirà nella lista delle Organization disponibili.

---

## 3) Creazione di un Team all’interno dell’Organization
Entrare nell’Organization `finance`, aprire la sezione **Teams** e creare un nuovo Team:

- **Team name:** `developers`  
- **Role:** `member`

Il Team sarà ora visibile nella lista.

---

## 4) Aggiunta di un utente al Team
All’interno del Team `developers`, aggiungere un membro:

- **Username:** `developer`

L’utente `developer` diventa ora parte del Team.

---

## 5) Creazione di un repository nell’Organization
All’interno dell’Organization `finance`, creare un nuovo repository:

- **Repository name:** `app`  
- **Visibility:** `private`

Il repository sarà inizialmente vuoto.

---

## 6) Assegnazione dei permessi al Team
Aprire il repository `finance/app`, entrare nella sezione **Settings → Permissions** e aggiungere:

- **Team:** `developers`  
- **Role:** `write`

Il Team `developers` ha ora il permesso di pushare immagini nel repository.

---

## 7) Push dell’immagine nel repository dell’Organization
Tornare al terminale ed eseguire il login come utente `developer` (se non già fatto).

Taggare l’immagine verso il repository dell’Organization:

```bash
podman tag alpine:latest registry.ocp4.example.com:8443/finance/app:latest
```

Eseguire il push:

```bash
podman push --tls-verify=false registry.ocp4.example.com:8443/finance/app:latest
```

Il push avrà successo perché l’utente `developer` appartiene al Team `developers`, che ha permesso `write`.

---

## 8) Verifica finale nella GUI
Accedere alla GUI e aprire:

```
Organization → finance → Repository → app → Tags
```

Dovrebbe essere visibile il tag `latest` appena pushato.

---

## Lab Quay - ambiente air-gapped


```bash
# ==============================================================================
# FASE 1: INSTALLAZIONE TOOL
# ==============================================================================

# Scaricamento del plugin oc-mirror (versione stable)
curl -L https://mirror.openshift.com/pub/openshift-v4/clients/ocp/stable/oc-mirror.tar.gz -o oc-mirror.tar.gz

# Estrazione dell'archivio
tar -xvzf oc-mirror.tar.gz

# Reso eseguibile e spostato nel PATH di sistema
chmod +x oc-mirror
sudo mv oc-mirror /usr/local/bin/

# ==============================================================================
# FASE 2: DEFINIZIONE IMAGESET (CONFIGURAZIONE)
# ==============================================================================

# Creazione del file dichiarativo ImageSetConfiguration
# Definisce lo scope: vogliamo solo l'immagine Alpine
cat <<EOF > imageset-config.yaml
kind: ImageSetConfiguration
apiVersion: mirror.openshift.io/v1alpha2
mirror:
  additionalImages:
    - name: docker.io/library/alpine:latest
EOF

# ==============================================================================
# FASE 3: DOWNLOAD (MIRROR TO DISK)
# ==============================================================================

# Esecuzione mirroring: Upstream -> Cartella Locale (usb-disk)
# Non richiede privilegi di admin, scarica solo i blob e manifest
oc mirror --v1 --config=imageset-config.yaml file://usb-disk

# ==============================================================================
# FASE 4: UPLOAD (DISK TO MIRROR)
# ==============================================================================

# Esecuzione mirroring: Cartella Locale -> Quay Interno
# Carica le immagini nell'org 'mirror-demo' e genera i manifest per il cluster
oc mirror --v1 --from ./usb-disk docker://registry.ocp4.example.com:8443/mirror-demo

# ==============================================================================
# FASE 5: CONFIGURAZIONE CLUSTER (REDIRECTION)
# ==============================================================================

# Entra nella cartella dei risultati generata automaticamente dall'upload
cd oc-mirror-workspace/results-*

# Applica la ImageContentSourcePolicy (ICSP) al cluster
# Istruisce CRI-O a reindirizzare docker.io -> registry.ocp4
oc apply -f imageContentSourcePolicy.yaml

# ==============================================================================
# FASE 6: VALIDAZIONE
# ==============================================================================

# Test: Richiesta esplicita dell'upstream originale (docker.io)
# Se il pod va in Running, il reindirizzamento trasparente funziona
oc run demo-test --image=docker.io/library/alpine:latest --restart=Never -- sleep 1000
```
