
# ZTP GitOps Demo – Redfish provisioning with Sushy Emulator

Questa demo mostra come **simulare un provisioning bare metal Redfish** utilizzando **Sushy Emulator** e una **VM KVM**, per validare il flusso ZTP (Zero Touch Provisioning) utilizzato da **ACM + Metal3** in OpenShift.

L’obiettivo è dimostrare:

* controllo Redfish (power / virtual media)
* integrazione Metal3
* readiness per un provisioning ZTP GitOps reale

---

## Prerequisiti

```bash
sudo dnf install -y podman libvirt qemu-kvm virt-install virt-viewer
```

### Apertura porte firewall

```bash
sudo firewall-cmd --add-port=8000/tcp --permanent
sudo firewall-cmd --reload
```

### Avvio libvirt

```bash
sudo systemctl enable --now libvirtd
```

---

## Creazione VM KVM (bare metal simulato)

```bash
sudo virt-install \
  --name worker-0 \
  --memory 512 \
  --vcpus 1 \
  --disk size=1,sparse=true \
  --os-variant generic \
  --network network=default \
  --pxe \
  --noreboot \
  --noautoconsole
```

Verifica VM:

```bash
sudo virsh list --all
```

Recupero UUID (usato da Redfish):

```bash
export VM_UUID=$(sudo virsh domuuid worker-0 | tr -d '\n')
```

---

## Avvio Sushy Emulator

### Script `start-sushy.sh`

Creare il seguente file `start-sushy.sh` nella cartella corrente.


```bash
#!/bin/bash

BASE_DIR="$(pwd)"
CONFIG_PATH="$BASE_DIR/sushy-emulator.conf"

if [ ! -f "$CONFIG_PATH" ]; then
    echo "Errore: File di configurazione non trovato in $CONFIG_PATH"
    exit 1
fi

container_exists=$(sudo podman ps -a --format "{{.Names}}" | grep -w sushy-tools)

if [ "$container_exists" ]; then
    container_running=$(sudo podman ps --format "{{.Names}}" | grep -w sushy-tools)
    if [ ! "$container_running" ]; then
        sudo podman start sushy-tools
    fi
else
    sudo podman run -d --privileged --rm --name sushy-tools \
        -v "$CONFIG_PATH:/etc/sushy/sushy-emulator.conf:Z" \
        -v /var/run/libvirt:/var/run/libvirt:Z \
        -e SUSHY_EMULATOR_CONFIG=/etc/sushy/sushy-emulator.conf \
        -p 8000:8000 \
        quay.io/metal3-io/sushy-tools:latest sushy-emulator
fi

echo "Sushy è attivo sulla porta 8000"
```


---

* Lo scopo del file è avviare Sushy Emulator come servizio Redfish persistente, collegato a libvirt, in modo idempotente (lo puoi rilanciare senza rompere nulla).
* E' necessario aggiungere anche il seguente file di configurazione: `sushy-emulator.conf`

### Configurazione `sushy-emulator.conf`

```python
SUSHY_EMULATOR_LISTEN_IP = u'0.0.0.0'
SUSHY_EMULATOR_LISTEN_PORT = 8000
SUSHY_EMULATOR_SSL_CERT = None
SUSHY_EMULATOR_SSL_KEY = None
SUSHY_EMULATOR_OS_CLOUD = None
SUSHY_EMULATOR_LIBVIRT_URI = u'qemu:///system'
SUSHY_EMULATOR_IGNORE_BOOT_DEVICE = True

SUSHY_EMULATOR_BOOT_LOADER_MAP = {
    u'UEFI': { u'x86_64': u'/usr/share/OVMF/OVMF_CODE.secboot.fd' },
    u'Legacy': { u'x86_64': None }
}

SUSHY_EMULATOR_VMEDIA_DEVICES = {
    "Cd": {
        "Name": "Virtual CD",
        "MediaTypes": ["CD", "DVD"]
    }
}
```

### Avvio sushy emulator:

```bash
chmod 755 start-sushy.sh
./start-sushy.sh
```

---

## Test Redfish API

Lista sistemi:

```bash
curl http://localhost:8000/redfish/v1/Systems/
```

Stato power:

```bash
curl -s http://localhost:8000/redfish/v1/Systems/$VM_UUID | grep -i PowerState
```

Spegnimento:

```bash
curl -X POST http://localhost:8000/redfish/v1/Systems/$VM_UUID/Actions/ComputerSystem.Reset \
  -H "Content-Type: application/json" \
  -d '{"ResetType": "ForceOff"}'
```

Accensione:

```bash
curl -X POST http://localhost:8000/redfish/v1/Systems/$VM_UUID/Actions/ComputerSystem.Reset \
  -H "Content-Type: application/json" \
  -d '{"ResetType": "On"}'
```

---

## Attach ISO via Virtual Media

Al momento la comunicazione è funzionante ma non c'è nessun sistema di Boot della VM. Aggiungiamo una ISO minimale:

```bash
curl -O https://dl-cdn.alpinelinux.org/alpine/v3.19/releases/x86_64/alpine-standard-3.19.1-x86_64.iso
sudo mv alpine-standard-3.19.1-x86_64.iso /var/lib/libvirt/images/
sudo chown qemu:qemu /var/lib/libvirt/images/alpine-standard-3.19.1-x86_64.iso
sudo restorecon -Rv /var/lib/libvirt/images/alpine-standard-3.19.1-x86_64.iso
```

Attach ISO:

```bash
sudo virt-xml worker-0 \
  --add-device \
  --disk /var/lib/libvirt/images/alpine-standard-3.19.1-x86_64.iso,device=cdrom,boot.order=1
```

Verifica boot:

```bash
virt-viewer --connect qemu:///system --wait worker-0
```

Login: `root` (no password)

---

## Attach ISO da HTTP ?

In alternativa si può acquisire l'immagine via HTTP in questo modo:

```bash
sudo virt-xml worker-0 \
  --add-device \
  --disk type=network,device=cdrom,protocol=https,\
name=/alpine/v3.19/releases/x86_64/alpine-standard-3.19.1-x86_64.iso,\
host.name=dl-cdn.alpinelinux.org,host.port=443,\
boot.order=1
```

## Test controllo Redfish da OpenShift

> Serve un cluster OpenShift attivo

```bash
oc login -u admin -p redhatocp https://api.ocp4.example.com:6443
```

> Verifica indirizzo IP workstation:

```bash
ip addr show
```

>Suggerimento: dovrebbe essere nel range 172.25.xx.xx

Esempio di Job di test (`job.yaml`):

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: test-vm-boot-curl
  namespace: openshift-machine-api
spec:
  template:
    spec:
      containers:
      - name: curl-boot
        image: curlimages/curl:latest
        command:
        - /bin/sh
        - -c
        - |
          curl -X POST http://IP:8000/redfish/v1/Systems/$VM_UUID/Actions/ComputerSystem.Reset \
            -H "Content-Type: application/json" \
            -d '{"ResetType": "On"}'
      restartPolicy: Never
  backoffLimit: 2
```

* Esegui Job:

```bash
oc create -f job.yaml
```
---

# Extra Steps - Creazione CR BareMetalHost

Metal3 e Ironic forniscono funzionalità di provisioning bare metal all’interno di OpenShift.
Ironic è il motore di provisioning sottostante che interagisce direttamente con i server fisici utilizzando protocolli di gestione standard come Redfish o IPMI per eseguire operazioni come il controllo dell’alimentazione, il boot via PXE e il deployment delle immagini.
Metal3 funge da livello di integrazione nativo Kubernetes sopra Ironic, esponendo queste capacità tramite Custom Resource come BareMetalHost e permettendo a OpenShift di gestire l’infrastruttura bare metal in modo dichiarativo, come parte di un flusso GitOps e Zero Touch Provisioning (ZTP).

Per questa parte utilizziamo una VM più dimensionata. Cancelliamo prima la precedente VM:

```bash
sudo virsh destroy worker-0
sudo virsh undefine worker-0 --nvram --remove-all-storage
```
- Creazione di una nuova VM

```yaml
sudo virt-install   --name worker-0   --memory 2048   --vcpus 1   --disk size=10,sparse=true   --os-variant rhel9.0   --network network=default,model=virtio,mac=52:54:00:00:00:01   --boot uefi   --noautoconsole   --noreboot   --check disk_size=off
```



La CR Provisioning serve per abilitare e configurare l’integrazione tra OpenShift, Metal3 e Ironic:

```bash
cat <<EOF | oc apply -f -
apiVersion: metal3.io/v1alpha1
kind: Provisioning
metadata:
  name: provisioning-configuration
spec:
  # 'Managed' dice all'operatore di installare Metal3 e Ironic
  provisioningStrategy: Managed
  watchAllNamespaces: true
  provisioningInterface: "eth0"
  provisioningNetwork: "Disabled"
EOF
```

Una volta creata questa risorsa, si può provare a fare il Provisioning: 

- Del Secret `secret.yaml` (credenziali non sono di fatto usate - quindi non occorre modificarle)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: worker-0-bmc-secret
  namespace: openshift-machine-api
type: Opaque
stringData:
  username: "admin"
  password: "password"
```

```bash
oc create -f secret.yaml
```

- Del BareMetalHost (sostituire IP, Mac Address e VM_UUID con valori reali)

- Recuperate il mac address della macchina:

```bash
sudo virsh domiflist worker-0
```

Recupero UUID (usato da Redfish):

```bash
export VM_UUID=$(sudo virsh domuuid worker-0 | tr -d '\n')
```


- Creazione del BareMetalHost (bmh.yaml):
- 
```yaml
apiVersion: metal3.io/v1alpha1
kind: BareMetalHost
metadata:
  name: demo-worker
  namespace: openshift-machine-api
  annotations:
    baremetalhost.metal3.io/inspect: "disabled"
spec:
  online: true
  bootMode: UEFI  
  bootMACAddress: "CHANGEME"
  bmc:
    address: redfish-virtualmedia+http://CHANGEME:8000/redfish/v1/Systems/VM_UUID
    credentialsName: worker-0-bmc-secret
    disableCertificateVerification: true
```

```bash
oc create -f bmh.yaml
```

# ZTP GitOps reale con SiteConfig (produzione)

Con hardware reale (o riutilizzando questa VM), il passo successivo è **non creare più manualmente il BareMetalHost**, ma usare **SiteConfig Operator** via GitOps.

CR da usare ( una delle due):

1) RAN Operator

```yaml
# 1. Crea il Namespace dedicato
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-topology-aware-lifecycle-manager
  labels:
    openshift.io/cluster-monitoring: "true" # Abilita il monitoraggio Prometheus

---
# 2. Definisce il Gruppo Operatori (Scope)
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: talm-operator-group
  namespace: openshift-topology-aware-lifecycle-manager
spec:
  targetNamespaces:
  - openshift-topology-aware-lifecycle-manager

---
# 3. La Subscription (L'installazione vera e propria)
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: topology-aware-lifecycle-manager
  namespace: openshift-topology-aware-lifecycle-manager
spec:
  # Il nome ufficiale del pacchetto su OLM
  name: topology-aware-lifecycle-manager
  # Dove trovarlo (Red Hat Catalog standard)
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  # Canale: usa "stable" o specifica la versione (es. "4.15", "4.16") allineata al tuo Hub
  channel: "stable"
  installPlanApproval: Automatic
```

2) SiteConfig Operator

```yaml
oc patch multiclusterhubs.operator.open-cluster-management.io \
  multiclusterhub -n ${MCH_NAMESPACE} \
  --type json \
  --patch '[{
    "op": "add",
    "path":"/spec/overrides/components/-",
    "value": {
      "name":"siteconfig",
      "enabled": true
    }
  }]'

```
---

## Flusso completo ZTP GitOps

```
Git commit
↓
ArgoCD sync
↓
SiteConfig reconciled
↓
InfraEnv generata
↓
Discovery ISO montata via Redfish
↓
Installazione OpenShift
↓
Cluster gestito da ACM
```

---

## Esempio SiteConfig per la VM gestita da Sushy

```yaml
apiVersion: ran.openshift.io/v1
kind: SiteConfig
metadata:
  name: sushy-demo-site
  namespace: ztp-sites
spec:
  baseDomain: example.com
  pullSecretRef:
    name: pull-secret
  sshPublicKey: "ssh-rsa AAAA..."

  clusters:
  - clusterName: sushy-cluster
    networkType: OVNKubernetes
    clusterLabels:
      common: true
    nodes:
    - hostName: virtual-worker-0
      role: master
      bmcAddress: redfish-virtualmedia://172.25.250.9:8000/redfish/v1/Systems/<VM_UUID>
      bmcCredentialsName:
        name: sushy-credentials
      bootMACAddress: 52:54:00:4e:45:cd
```

Settaggi principali da cambiare:

* `bmcAddress`
* `bmcCredentials`
* hardware fisico al posto della VM



