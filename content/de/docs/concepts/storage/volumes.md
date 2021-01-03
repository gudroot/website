<!-- overview -->

Daten auf Disk sind in einem Container flüchtig, was für nicht-triviale Anwendungen ein Problem darstellt, wenn sie in Containern laufen sollen. Ein Problem ist der Verlust von Dateien, wenn der Container crasht. Das kubelet startet den Container neu, aber in einem clean state. Ein zweites Problem tritt auf, wenn Dateien zwischen Containern geteilt werden sollen, die zusammen in einem `Pod` laufen. Die Abstraktion des Kubernetes {{< glossary_tooltip text="volume" term_id="volume" >}} löst beide Probleme. Vertrautheit mit [Pods](/docs/concepts/workloads/pods/) wird vorausgesetzt.

<!-- body -->

## Hintergrund

Docker hat ein Konzept von [Volumes](https://docs.docker.com/storage/), aber es ist etwas lockerer und weniger *managed*(?). Ein Docker-Volume ist ein Verzeichnis auf Platte oder in einem anderen Container. Docker stellt Volume-Treiber zur Verfügung, aber die Funktionalität ist etwas eingeschränkt.

Kubernetes unterstützt viele Typen von Volumes. Ein {{< glossary_tooltip term_id="pod" text="Pod" >}} kann eine bliebige Anzahl von Volume-Types gleichzeitig nutzen. Flüchtige Volume-Types haben die Lebensdauer eines Pods, aber persistente Volumes existieren über die Lebensdauer eines Pods hinaus. Folglich überlebt ein Volume alle Container, die in einem Pod laufen, und die Daten werden über Conatiner-Restarts hinaus bewahrt. Wen ein Pod aufhört zu existieren, wird das Volume zerstört.

Im Kern ist ein Volume nur ein Verzeichnis, möglicherweise mit ein paar Daten darin, das für die Container in einem Pod zugänglich ist. Wie dieses Verzeichnis entsteht, welches Medium dahinter steht und sein Inhalt werden vom jeweiligen Volume-Typ festgelegt.

Um ein Volume zu nutzen, spezifiziere die Volumes, die dem Pod zur Verfügung gestellt werden sollen, in `.spec.volumes` und gib in `.spec.containers[*].volumeMounts` an wo diese Volumes in den Containern gemounted werden sollen. Ein Prozess in einem Container sieht aus dem Docker Image und den Volumes zusammengesetzte Sicht des Dateisystems. Das [Docker image](https://docs.docker.com/userguide/dockerimages/) bildet die Wurzel der Filesystem-Hierarchie. Volumes werden an den angegebenen Pfaden im Image eingehängt. Volumes können nicht in andere Volumes eingehängt werden oder hard links zu anderen Volumes haben. Jeder Container in der Konfiguration des Pods muss für sich spezifizieren, wo jedes Volume gemounted werden soll.

## Volume-Types {#volume-types}

Kubernetes unterstützt mehrere Typen von Volumes.

### awsElasticBlockStore {#awselasticblockstore}

Ein `awsElasticBlockStore` Volume mounted ein Amazon Web Services (AWS)
[EBS volume](https://aws.amazon.com/ebs/) in deinen Pod. Anders als `emptyDir`, das gelöscht wird, wenn der Pod entfernt wird, bleibt der Inhalt eines EBS Volumes erhalten und das Volume wird lediglich unmounted. Das bedeutet, dass ein EBS Volume vorab mit Daten gefüllt werden kann und dass Daten zwischen Pods geteilt werden können.

{{< note >}}
Ein EBS Volume muss mit `aws ec2 create-volume` oder dem AWS API erstellt werden bevor man es nutzen kann.
{{< /note >}}

Es gibt ein paar Einschränkungen, wenn man ein `awsElasticBlockStore` nutzt:
* die Nodes, auf denen die Pods laufen, müssen AWS EC2 Instanzen sein
* diese Instanzen müssen in der selben Region und Availability Zone sein wie das EBS Volume
* EBS kann nur von einer einzigen EC2 Instanz gemounted werden

#### Ein AWS EBS Volume erstellen

Bevor du ein EBS Volume in einem Pod nutzen kannst, musst du es erstellen.

```shell
aws ec2 create-volume --availability-zone=eu-west-1a --size=10 --volume-type=gp2
```

Stelle sicher, dass die Zone der Zone entsprocht, in der du den Cluster erstellt hast. Schau auch, dass die Grösse und der EBS Volume Typ für deinen Zweck passt.

#### AWS EBS Beispielkonfiguration

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-ebs
spec:
  containers:
  - image: k8s.gcr.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /test-ebs
      name: test-volume
  volumes:
  - name: test-volume
    # This AWS EBS volume must already exist.
    awsElasticBlockStore:
      volumeID: "<volume id>"
      fsType: ext4
```

#### AWS EBS CSI migration

{{< feature-state for_k8s_version="v1.17" state="beta" >}}

Das `CSIMigration` Feature von `awsElasticBlockStore`, leitet, wenn es eingeschaltet ist, alle Plugin-Operationen vom existierenden in-tree Plugin zum `ebs.csi.aws.com` Container Storage Interface (CSI) Driver weiter. Um dieses Feature nutzen zu können, muss der [AWS EBS CSI driver](https://github.com/kubernetes-sigs/aws-ebs-csi-driver) installiert sein und die Beta-Features `CSIMigration` and `CSIMigrationAWS` müssen eingeschaltet sein.

#### AWS EBS CSI migration complete

{{< feature-state for_k8s_version="v1.17" state="alpha" >}}

Um zu verhindern, dass das `awsElasticBlockStore` Storage Plugin vom Controller Manager und vom Kubelet geladen wird, setze das `CSIMigrationAWSComplete` Flag auf `true`. Dieses Feature setzt voraus, dass der `ebs.csi.aws.com` Container Storage Interface (CSI) Driver auf allen Worker Nodes installiert ist.

### azureDisk {#azuredisk}

Der `azureDisk` Volume-Typ mounted eine Microsoft Azure [Data Disk](https://docs.microsoft.com/en-us/azure/aks/csi-storage-drivers) in einen Pod.

Zu den Details vgl. [`azureDisk` volume plugin](https://github.com/kubernetes/examples/tree/{{< param "githubbranch" >}}/staging/volumes/azure_disk/README.md).

#### azureDisk CSI migration

{{< feature-state for_k8s_version="v1.19" state="beta" >}}

Das `CSIMigration` Feature von `azureDisk`, leitet, wenn es eingeschaltet ist, alle Plugin-Operationen vom existierenden in-tree Plugin zum `disk.csi.azure.com` Container Storage Interface (CSI) Driver weiter. Um dieses Feature nutzen zu können, muss der [Azure Disk CSI
Driver](https://github.com/kubernetes-sigs/azuredisk-csi-driver) installiert sein und die Features `CSIMigration` and `CSIMigrationAzureDisk` müssen eingeschaltet sein.

### azureFile {#azurefile}

Der `azureFile` Volume-Typ mounted ein Microsoft Azure File volume (SMB 2.1 and 3.0) in einen Pod.

Zu den Details vgl. [`azureFile` volume plugin](https://github.com/kubernetes/examples/tree/{{< param "githubbranch" >}}/staging/volumes/azure_file/README.md).

#### azureFile CSI migration

{{< feature-state for_k8s_version="v1.15" state="alpha" >}}

Das `CSIMigration` Feature von `azureFile`, leitet, wenn es eingeschaltet ist, alle Plugin-Operationen vom existierenden in-tree Plugin zum `file.csi.azure.com` Container Storage Interface (CSI) Driver weiter. Um dieses Feature nutzen zu können, muss der [Azure File CSI
Driver](https://github.com/kubernetes-sigs/azurefile-csi-driver) installiert sein und die Alpha-Features `CSIMigration` and `CSIMigrationAzureFile` müssen eingeschaltet sein.

### cephfs

Ein `cephfs` Volume erlaubt es ein existierendes CephFS Volume in deinen Pod zu mounten. Anders als `emptyDir`, das gelöscht wird, wenn der Pod entfernt wird, bleibt der Inhalt eines EBS Volumes erhalten und das Volume wird lediglich unmounted. Das bedeutet, dass ein EBS Volume vorab mit Daten gefüllt werden kann und dass Daten zwischen Pods geteilt werden können. Das `cephfs` Volume kann von mehreren Schreibern simultan gemounted werden.

{{< note >}}
Du musst einen eigenen Ceph-Server laufen und den Share exportiert haben bevor du ihn nutzen kannst.
{{< /note >}}

Vgl. [CephFS example](https://github.com/kubernetes/examples/tree/{{< param "githubbranch" >}}/volumes/cephfs/) für mehr Details.

### cinder

{{< note >}}
Kubernetes muss mit dem OpenStack Cloud-Provider konfiguriert werden.
{{< /note >}}

Der `cinter` Volume-Typ wird verwendet um ein OpenStack Cinder Volume in einen Pod zu mounten.

#### Cinder volume Beispielskonfiguration

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-cinder
spec:
  containers:
  - image: k8s.gcr.io/test-webserver
    name: test-cinder-container
    volumeMounts:
    - mountPath: /test-cinder
      name: test-volume
  volumes:
  - name: test-volume
    # This OpenStack volume must already exist.
    cinder:
      volumeID: "<volume id>"
      fsType: ext4
```

#### OpenStack CSI migration

{{< feature-state for_k8s_version="v1.18" state="beta" >}}

Das `CSIMigration` Feature von Cinder, leitet, wenn es eingeschaltet ist, alle Plugin-Operationen vom existierenden in-tree Plugin zum `cinder.csi.openstack.org` Container Storage Interface (CSI) Driver weiter. Um dieses Feature nutzen zu können, muss der [Openstack Cinder CSI
Driver](https://github.com/kubernetes/cloud-provider-openstack/blob/master/docs/using-cinder-csi-plugin.md) installiert sein und die Beta-Features `CSIMigration` and `CSIMigrationOpenStack` müssen eingeschaltet sein.

### configMap

Eine [ConfigMap](/docs/tasks/configure-pod-container/configure-pod-configmap/) stellt einen Weg dar Konfigurationsdaten in einen Pod zu füttern. Die Daten, die in einer ConfigMap gespeichert sind, können in einem Volume des Typs `configMap` referenziert werden und dann von containerisierten Applikationen in einem Pod konsumiert werden.

Wenn man eine ConfigMap referenziert, gibt man den Namen der ConfigMap im Volume an. Man kann den für einen spezifischen Eintrag in der ConfigMap anpassen. Die folgende Konfiguration zeigt wie man die `log-config` ConfigMap in einem Pod namens `configmap-pod` mounted:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-pod
spec:
  containers:
    - name: test
      image: busybox
      volumeMounts:
        - name: config-vol
          mountPath: /etc/config
  volumes:
    - name: config-vol
      configMap:
        name: log-config
        items:
          - key: log_level
            path: log_level
```

Die `log-confg` ConfigMap wird als Volume gemounted und der in ihrem `log_level` Eintrag gespeicherte Content wird in den Pod unter dem Pfad `/etc/config/log_level` gemounted. Beachte, dass sich dieser Pfad aus dem `mountPath` des Volumes und dem als `log_level` angegebenen `path` zusammensetzt.

{{< note >}}
* Du musst eine [ConfigMap](/docs/tasks/configure-pod-container/configure-pod-configmap/) erstellen bevor du sie nutzen kannst.

* Ein Container der eine ConfigMap als [`subPath`](#using-subpath) Volume Mount nutzt, wird Änderungen an der ConfigMap nicht mitkriegen.

* Text-Daten werden als Dateien in UTF-8 Encoding zur Verfügung gestellt. Verwende `binaryData` für andere Encodings.
{{< /note >}}

### downwardAPI {#downwardapi}

Ein `downwardAPI` Volume macht downward API Daten für Applikationen zugänglich. Es mounted ein Verzeichnis und schreibt die verlangten Daten in plain text Files.

{{< note >}}
Ein Container, der das downward API als [`subPath`](#using-subpath) Volume Mount nutzt, wird Updates des downward APIs nicht mitkriegen.
{{< /note >}}

Zu den Details vgl. [downward API example](/docs/tasks/inject-data-application/downward-api-volume-expose-pod-information/).

### emptyDir {#emptydir}

Ein `emptyDir` Volume wird erstellt, wenn ein Pod einem Node zugewiesen wird und existiert so lange wie der Pod auf diesem Node läuft. Wie der Name sagt, ist das `emptyDir` Volume anfangs leer. Alle Container im Pod können die selben Dateien im `emptyDir` Volume lesen und schreiben, doch das Volume kann am selben oder auch an unterschiedlichen Orten in jedem Container gemounted werden. Wenn der Pod aus irgendeinem Grund von einem Node entfernt wird, werden die Daten im `emptyDir` permanent gelöscht.

{{< note >}}
Ein gecrashter Container entfernt einen Pod *nicht* von einem Node. Die Daten in einem `emptyDir` Volume überstehen Container Crashes.
{{< /note >}}

Anwendungsfälle für ein `emptyDir` sind:
* scratch space, z.B. für einen disk-based merge sort
* Checkpoints in einer langen Berechnung für die Wiederherstellung nach einem Crash
* das Halten von Daten, die ein Content-Manager holt, während ein Webserver-Container die Daten serviert.

Abhängig von deinem Environment werden `emptyDir` Volumes auf dem Medium gespeichert, auf dem die Nodes liegen, wie etwa Festplatten oder SSD, oder auf Netzwerk-Storage. Wenn du das `emptyDir.medium` Feld auf `"Memory"` setzt, mounted Kubernetes stattdessen ein tmpfs (RAM-backed filesystem) für dich. tmpfs ist zwar sehr schnell, aber denke daran, dass tmpfs anders als Platten beim Reboot eines Nodes geleert wird und die Daten, die du schreibst, zum Memory Limit deines Containers gezählt werden.

{{< note >}}
Wenn das `SizeMemoryBackedVolumes` [feature gate](/docs/reference/command-line-tools-reference/feature-gates/) aktivert ist, kann die Grösse von memory backed Volumes angegeben werden. Wenn keine Grösse angegeben wird, wird die Grösse von memory backed Volumes auf einem Linux Host mit 50% des Speichers festgelegt.
{{< /note>}}

#### emptyDir Beispielskonfiguration

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: k8s.gcr.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /cache
      name: cache-volume
  volumes:
  - name: cache-volume
    emptyDir: {}
```

### fc (fibre channel) {#fc}

Der `fc` Volume-Typ erlaubt es ein existierendes Fibre Channel Block Storage Volume in einen Pod zu mounten. Man kann einzelne oder mehrere target world wide names (WWNs) im Parameter `targetWWNs` in der Volume Konfiguration angeben. Wenn mehrere WWNs angegeben werden, erwartet targetWWNs, dass diese WWNs von multi-path Verbindungen kommen.

{{< note >}}
You must configure FC SAN Zoning to allocate and mask those LUNs (volumes) to the target WWNs
beforehand so that Kubernetes hosts can access them.
{{< /note >}}

Zu den Details vgl. [fibre channel example](https://github.com/kubernetes/examples/tree/{{< param "githubbranch" >}}/staging/volumes/fibre_channel).

### flocker (deprecated) {#flocker}

[Flocker](https://github.com/ClusterHQ/flocker) ist ein open-source clustered container data volume manager. Flocker stellt ein Management und die Orchestrierung von Daten-Volumes auf einer Reihe von Storage Backends zur Verfügung.

Ein `flocker` Volume erlaubt es ein Flocker dataset in einen Pod zu mounten. Wenn das dataset noch nicht existiert, muss es zuerst mit dem Flocker CLI oder über das Flocker API erstellt werden. Wenn das dataset schon existiert, wird es von Flocker auf dem Node, auf dem der Pod gescheduled wird, eingehängt. Dies bedeutet, dass Daten zwischen Pods nach Bedarf geteilt werden können.

{{< note >}}
Du musst deine eigene Flocker Installation laufen haben bevor du sie nutzen kannst.
{{< /note >}}

Zu den Details vgl. [Flocker example](https://github.com/kubernetes/examples/tree/{{< param "githubbranch" >}}/staging/volumes/flocker).

### gcePersistentDisk

Ein `gcePersistentDisk` Volume mountet eine Google Compute Engine (GCE)
[persistent disk](https://cloud.google.com/compute/docs/disks) (PD) in deinen Pod. Anders als `emptyDir`, das gelöscht wird, wenn ein Pod entfernt wird, bleibt der Inhalt einer PD erhalten und das Volume wird lediglich unmounted.
into your Pod. Das bedeutet, dass eine PD vorab mit Daten gefüllt werden kann und dass Daten zwischen Pods geteilt werden können. 

{{< note >}}
Du musst eine PD mit `gcloud` oder über das GCE API oder UI erstellen bevor du sie nutzen kannst.
{{< /note >}}

Es gibt ein paar Einschränkungen, wenn man eine `gcePersistentDisk` verwendet:
* die Nodes, auf denen die Pods laufen, müssen GCE VMs sein
* diese VMs müssen im selben GCE Projekt und der selben Zone sein wie die persistent disk.

Ein Feature von GCE persistent disk ist gleichzeitiger read-only Zugriff auf eine persistent disk. Ein `gcePeresistentDisk` Volume erlaubt es mehreren Consumern eine persistent disk gleichzeitig read-only zu mounten. Das bedeutet, dass man eine PD schon vorab mit einem Datenset bestücken kann und sie dann parallel an so viele Pods servieren wie nötig. Leider können PDs nur von einem einzelnen Consumer im read-write Modus gemounted werden. Gleichzeitige Schreibzugriffe sind nicht erlaubt.

Eine GCE persistent disk mit einem Pod zu verwenden, der von einem ReplicaSet kontrolliert wird, wird fehlschlagen, wenn die PD nicht read-only ist oder die replica count 0 oder 1 beträgt.

#### Erstellen einer GCE persistent disk {#gce-create-persistent-disk}

Bevor du eine GCE persistent disk verwenden kannst, musst du sie erstellen.

```shell
gcloud compute disks create --size=500GB --zone=us-central1-a my-data-disk
```

#### GCE persistent disk Beispielskonfiguration

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: k8s.gcr.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /test-pd
      name: test-volume
  volumes:
  - name: test-volume
    # This GCE PD must already exist.
    gcePersistentDisk:
      pdName: my-data-disk
      fsType: ext4
```

#### Regional persistent disks

Das [Regional persistent disks](https://cloud.google.com/compute/docs/disks/#repds) Feature erlaubt die Erstellung von persistent disks, die in zwei Zonen innerhalb der selben Region verfügbar sind. Um dieses Feature nutzen zu können muss das Volume als PersistentVolume provisioniert werden; das Volume direkt aus einem Pod zu referenzieren ist nicht unterstützt.

#### Händisches Provisionieren eines Regional PD PersistentVolume

Dynamisches Provisionieren ist möglich mit [StorageClass for GCE PD](/docs/concepts/storage/storage-classes/#gce).
Bevor man ein PersistentVolume anlegen kann, muss man die persistent disk anlegen:

```shell
gcloud compute disks create --size=500GB my-data-disk
  --region us-central1
  --replica-zones us-central1-a,us-central1-b
```

#### Regional persistent disk Beispielskonfiguration

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: test-volume
spec:
  capacity:
    storage: 400Gi
  accessModes:
  - ReadWriteOnce
  gcePersistentDisk:
    pdName: my-data-disk
    fsType: ext4
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: failure-domain.beta.kubernetes.io/zone
          operator: In
          values:
          - us-central1-a
          - us-central1-b
```

#### GCE CSI migration

{{< feature-state for_k8s_version="v1.17" state="beta" >}}

Das `CSIMigration` Feature von GCE PD leitet, wenn es eingeschaltet ist, alle Plugin-Operationen vom existierenden in-tree Plugin zum `pd.csi.storage.gke.io` Container Storage Interface (CSI) Driver weiter. Um dieses Feature nutzen zu können, muss der [GCE PD CSI
Driver](https://github.com/kubernetes-sigs/gcp-compute-persistent-disk-csi-driver) installiert sein und die Beta-Features `CSIMigration` and `CSIMigrationGCE` müssen eingeschaltet sein.

### gitRepo (deprecated) {#gitrepo}

{{< warning >}}
Der Volume-Typ `gitRepo` ist deprecated. Um ein Git Repo in einem Container zu provisionieren mounte ein [EmptyDir](#emptydir) in einen InitContainer, der das Repo mit git klont und danach mounte das [EmptyDir](#emptydir) in den Container des Pods.
{{< /warning >}}

Ein `gitRepo` Volume ist ein Beispiel für ein Volume Plugin. Dieses Plugin mounted ein leeres Verzeichnis und klont ein Git Repository in dieses Verzeichnis damit dein Pod es nutzen kann.

Hier ist ein Beispiel für ein `gitRepo` Volume:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: server
spec:
  containers:
  - image: nginx
    name: nginx
    volumeMounts:
    - mountPath: /mypath
      name: git-volume
  volumes:
  - name: git-volume
    gitRepo:
      repository: "git@somewhere:me/my-git-repository.git"
      revision: "22f1d8406d464b0c0874075539c1f2e96c253775"
```

### glusterfs

Ein `glusterfs` Volume erlaubt es ein [Glusterfs](https://www.gluster.org) (ein open source Netzerk-Dateisystem) Volume in einen Pod zu mounten. Anders als `emptyDir`, das gelöscht wird, wenn ein Pod entfernt wird, bleibt der Inhalt eines `glusterfs` Volumes erhalten und das Volume wird lediglich unmounted.
into your Pod. Das bedeutet, dass ein glusterfs Volume vorab mit Daten gefüllt werden kann und dass Daten zwischen Pods geteilt werden können.  GlusterFs kann von mehreren Schreibern zugleich gemounted werden.

{{< note >}}
Du musst deine eigene Gluster FS Installation laufen haben bevor du sie nutzen kannst.
{{< /note >}}

Zu den Details vgl. [GlusterFS example](https://github.com/kubernetes/examples/tree/{{< param "githubbranch" >}}/volumes/glusterfs).

### hostPath {#hostpath}

Ein `hostPath` Volume mountet eine Datei oder ein Verzeichnis aus dem Dateisystem des Nodes in deinen Pod. Das ist nichts, was die meisten Pods brauchen werden, aber es öffnet einige mächtige Möglichkeiten für manche Applikationen.

Einige Anwendungsbeispiele für einen `hostPath`:

* einen Container laufen lassen, der Zugriff auf Docker-Interna braucht; verwende einen `hostPath` von `/var/lib/docker`
* cAdvisor in einem Container laufen lassen; verwende einen `hostPath` von `/sys`
* einem Pod erlauben anzugeben, ob ein gegebener `hostPath` schon existieren soll bevor der Pod läuft, ob er angelegt werden soll und in welcher Form er existieren soll.

Zusätzlich zur verpflichtenden Eigenschaft `path` kann man optional auch einen `type` für ein `hostPath` Volume angeben.

Die unterstützten Werte des Feldes `type` sind:

| Wert | Verhalten |
| | Leerer String (default) dient der Abwärtskompatibilität und bedeutet, dass keine Checks durchgeführt werden bevor das hostPath Volume gemounted wird. |
| `DirectoryOr Create` | Wenn am angegebenen Pfad nichts existiert, wird ein leeres Verzeichnis angelegt und die Permissions auf 0755 gesetzt mit der selben Gruppe und dem selben Owner wie das Kubelet. |
| `Directory` | Am angegebenen Pfad muss ein Verzeichnis existieren. |
| `FileOrCreate` | Wenn am angegebenen Pfad nichts existiert, wird eine leere Datei angelegt mit den Permissions 0664 und der selben Gruppe und dem selben Owner wie das Kubelet. |
| `File` | Am angegebenen Pfad muss eine Datei existieren. |
| `Socket` | Am angegebenen Pfad muss ein UNIX socket existieren. |
| `CharDevice` | Am angegebenen Pfad muss ein character device existieren. |
| `BlockDevice` | Am angegebenen Pfad muss ein block device existieren. |

Sei vorsichtig, wenn du diesen Volume Typ verwendest, weil:
* Pods mit identischer Konfiguration (z.B. von einem PodTemplate erstellt) könnens ich auf unterschiedlichen Nodes unterschiedlich verhalten, weil sich Dateien auf den Nodes unterscheiden.
* Die Dateien und Verzeichnisse auf den darunterliegenden Hosts sind nur für Root schreibbar. Du musst entweder deinen Prozess als Root in einem privilegierten Container laufen lassen oder die Dateiberechtigungen auf dem Host ändern um auf ein `hostPath` Volume schreiben zu können.

#### hostPath Beispielskonfiguration

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: k8s.gcr.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /test-pd
      name: test-volume
  volumes:
  - name: test-volume
    hostPath:
      # directory location on host
      path: /data
      # this field is optional
      type: Directory
```

{{< caution >}}
Der `FileOrCreate` Modus erstellt das Parent-Directory der Datei nicht. Wenn das Parent-Directory der gemounteten Datei nicht existiert, kann der Pod nicht starten. Um sicherzustellen, dass dieser Modus funktioniert, kannst du versuchen Verzeichnisse und Dateien separat zu mounten, wie es in [`FileOrCreate`configuration](#hostpath-fileorcreate-example) gezeigt wird.
{{< /caution >}}

#### hostPath FileOrCreate Beispielskonfiguration {#hostpath-fileorcreate-example}

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-webserver
spec:
  containers:
  - name: test-webserver
    image: k8s.gcr.io/test-webserver:latest
    volumeMounts:
    - mountPath: /var/local/aaa
      name: mydir
    - mountPath: /var/local/aaa/1.txt
      name: myfile
  volumes:
  - name: mydir
    hostPath:
      # Ensure the file directory is created.
      path: /var/local/aaa
      type: DirectoryOrCreate
  - name: myfile
    hostPath:
      path: /var/local/aaa/1.txt
      type: FileOrCreate
```

### iscsi

Ein `iscsi` Volume erlaubt es ein existierendes iSCSI (SCSI over IP) Volume in einen Pod zu mounten. Anders als `emptyDir`, das gelöscht wird, wenn ein Pod entfernt wird, bleibt der Inhalt eines `iscsi` Volumes erhalten und das Volume wird lediglich unmounted. Das bedeutet, dass ein iscsi Volume vorab mit Daten gefüllt werden kann und dass Daten zwischen Pods geteilt werden können. 

{{< note >}}
Du musst deinen eigenen iSCSI Server laufen haben und das Volume erstellt haben bevor du es nutzen kannst.
{{< /note >}}

Ein Feature von iSCSI ist gleichzeitiges read-only Mounten durch mehrere Consumer. Das bedeutet, dass man ein Volume schon vorab mit einem Datenset bestücken kann und sie dann parallel an so viele Pods servieren wie nötig. Leider können iSCSI Volumes nur von einem einzelnen Consumer im read-write Modus gemounted werden. Gleichzeitige Schreibzugriffe sind nicht erlaubt.

Zu den Details vgl. [iSCSI example](https://github.com/kubernetes/examples/tree/{{< param "githubbranch" >}}/volumes/iscsi).

### local

Ein `local` Volume repräsentiert ein gemountetes lokales Storage-device wie eine Disk, Partition oder ein Verzeichnis.

Local Volumes können nur als statisch erstellte PersistentVolumes verwendet werden. Dynamisches Provisionieren ist nicht unterstützt.

Verglichen mit `hostPath` Volumes werden `local` Volumes dauerhaft und portabel verwendet ohne manuell Pods auf Nodes zu schedulen. Das System weiss von den Node Constraints des Volumes indem es auf die Node Affinity des PersistentVolumes schaut.

Dennoch sind `local` Volumes von der Verfügbarkeit des darunterliegenden Nodes ab und sind nicht für alle Applikationen geeignet. Wenn ein Node "unhealthy" wird, wird das `local` Volume für den Pod unzugänglich. Der Pod, der dieses Volume vewendet, kann nicht laufen. Applikationen, die `local` Volumes verwenden, müssen diese reduzierte Verfügbarkeit aushalten können, ebenso wie den potentiellen Verlust an Daten, abhängig von der Beständigkeit der darunterliegenden Platte.

Das folgende Beispiel zeigt ein PersistentVolume, das ein `local` Volume und `nodeAffinity` verwendet:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-pv
spec:
  capacity:
    storage: 100Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /mnt/disks/ssd1
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - example-node
```

Du musst eine `nodeAffinity` beim PersistentVolume setzen, wenn du `local` Volumes verwendest. Der Kubernetes Scheduler verwendet die `nodeAffinity` des PersistentVolume um diese Pods auf den korrekten Node zu schedulen.

Der `volumeMode` des PersistentVolume kann auf "Block" (anstelle des default Wertes "Filesystem") gesetzt werden um das lokale Volume als raw block device zu exposen.

Wenn man lokale Volumes verwendet, ist es empfohlen eine StorageClass zu erstellen, die den `volumeBindingMode` auf `WaitForFirstConsumer` gesetzt hat. Für mehr Details vgl. das local [StorageClass](/docs/concepts/storage/storage-classes/#local) Beispiel. Das Volume Binding zu verschieben stellt sicher, dass die PersistentVolumeClaim Binding Entscheidung zusammen mit anderen Node Constraints evaluiert werden wird, die der Pod haben mag, wie etwa Anforderungen an Node Resourcen, Node Selectors, Pod Affinity und Pod Anti-Affinity.

Man kann einen externen static Provisioner separat laufen lassen um das Management des local Volume Lifecycles zu verbessern. Merke, dass dieser Provisioner noch kein dynamisches Provisionieren unterstützt. Für ein Beispiel wie man einen externen local Provisioner betreibt vgl. den [local volume provisioner user
guide](https://github.com/kubernetes-sigs/sig-storage-local-static-provisioner).

{{< note >}}
Local PersistentVolume benötigt manuelles Aufräumen und Löschen durch den User wenn der externe static Provisioner nicht verwendet wird um den Volume Lifecycle zu managen.
{{< /note >}}

### nfs

Ein `nfs` Volume erlaubt es ein existierendes NFS (Network File System) Share in einen Pod zu mounten. Anders als `emptyDir`, das gelöscht wird, wenn ein Pod entfernt wird, bleibt der Inhalt eines `nfs` Volumes erhalten und das Volume wird lediglich unmounted. Das bedeutet, dass ein NFS Volume vorab mit Daten gefüllt werden kann und dass Daten zwischen Pods geteilt werden können. NFS kann von mehreren schreibenden Pods gleichzeitig gemounted werden.

{{< note >}}
Du musst deinen eigenen NFS Server laufen und den Share exportiert haben bevor du ihn nutzen kannst.
{{< /note >}}

Für Details vgl. das [NFS example](https://github.com/kubernetes/examples/tree/{{< param "githubbranch" >}}/staging/volumes/nfs).

### persistentVolumeClaim {#persistentvolumeclaim}

Ein `persistentVolumeClaim` Volume wird verwendet um ein [PersistentVolume](/docs/concepts/storage/persistent-volumes/) in einen Pod zu mounten. PersistentVolumeClaims sind ein Weg für User dauerhaftes Storage (wie z.B. eine GCE PersistentDisk oder ein iSCSI Volume) anzufordern ohne die Details des jeweiligen Cloud-Environments zu kennen.

Zu den Details vgl. die Information zu [PersistentVolumes](/docs/concepts/storage/persistent-volumes/).

### portworxVolume {#portworxvolume}

Ein `portworxVolume` ist ein elastic block storage Layer, der hyperconverged mit Kubernetes läuft.  [Portworx](https://portworx.com/use-case/kubernetes-storage/) erstellt Fingerprints des Storages in einem Server, erstellt Tiers basieren auf dessen Leistungsumfang und aggregiert die Kapazität über mehrere Server. Portworx läuft als Guest in virtuellen Maschinen oder auf bare metal Linux Nodes.

Ein `portworxVolume` kann dynamisch durch Kubernetes erstellt werden, aber es kann auch vorprovisioniert und in einem Pod referenziert werden.

Hier ist ein Beispiel für einen Pod, der ein vorprovisioniertes Portworx Volume referenziert:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-portworx-volume-pod
spec:
  containers:
  - image: k8s.gcr.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /mnt
      name: pxvol
  volumes:
  - name: pxvol
    # This Portworx volume must already exist.
    portworxVolume:
      volumeID: "pxvol"
      fsType: "<fs-type>"
```

{{< note >}}
Stelle sicher, dass du ein existierendes PortworxVolume mit dem Namen `pxvol` hast bevor du es im Pod verwendest.
{{< /note >}}

Zu den Details vgl. [Portworx volume](https://github.com/kubernetes/examples/tree/{{< param "githubbranch" >}}/staging/volumes/portworx/README.md).

### projected

Ein `projected` Volume mappt mehrere existierende Volume Quellen in das selbe Directory.

Derzeit können die folgenden Volume Quellen zusammengefasst werden:

* [`secret`](#secret)
* [`downwardAPI`](#downwardapi)
* [`configMap`](#configmap)
* `serviceAccountToken`

Alle Quellen müssen im selben Namespace sein wie der Pod. Zu den Details vgl. [all-in-one volume design document](https://github.com/kubernetes/community/blob/{{< param "githubbranch" >}}/contributors/design-proposals/node/all-in-one-volume.md).

#### Beispielskonfiguration mit einem Secret, einer downwardAPI und einer ConfigMap {#example-configuration-secret-downwardapi-configmap}

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-test
spec:
  containers:
  - name: container-test
    image: busybox
    volumeMounts:
    - name: all-in-one
      mountPath: "/projected-volume"
      readOnly: true
  volumes:
  - name: all-in-one
    projected:
      sources:
      - secret:
          name: mysecret
          items:
            - key: username
              path: my-group/my-username
      - downwardAPI:
          items:
            - path: "labels"
              fieldRef:
                fieldPath: metadata.labels
            - path: "cpu_limit"
              resourceFieldRef:
                containerName: container-test
                resource: limits.cpu
      - configMap:
          name: myconfigmap
          items:
            - key: config
              path: my-group/my-config
```

#### Beispielskonfiguration: Secrets mit non-Standard Berechtigungen {#example-configuration-secrets-nondefault-permission-mode}

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-test
spec:
  containers:
  - name: container-test
    image: busybox
    volumeMounts:
    - name: all-in-one
      mountPath: "/projected-volume"
      readOnly: true
  volumes:
  - name: all-in-one
    projected:
      sources:
      - secret:
          name: mysecret
          items:
            - key: username
              path: my-group/my-username
      - secret:
          name: mysecret2
          items:
            - key: password
              path: my-group/my-password
              mode: 511
```

Jede Volume Quelle wird in der Spec unter `sources` aufgelistet. Die Parameter sind fast die selben mit zwei Ausnahmen:

* Bei Secrets muss deas `secretName` Feld auf `name` geändert werden um mit der ConfigMap Konvention konsistent zu sein.
* der `defaultMode` kann nur auf der Ebene der Zusammenfassung angegeben werden und nicht für jede Volume Quelle extra. Du kannst aber, wie oben dargestellt, den `mode` für jedes einzelne Element angeben.

Wenn das `TokenRequestProjection` Feature eingeschaltet ist, kannst du das Token fur den gerade verwendeten [service account](/docs/reference/access-authn-authz/authentication/#service-account-tokens) an einem angegebenen Pfad in einen Pod einfügen. Zum Beispiel:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sa-token-test
spec:
  containers:
  - name: container-test
    image: busybox
    volumeMounts:
    - name: token-vol
      mountPath: "/service-account"
      readOnly: true
  volumes:
  - name: token-vol
    projected:
      sources:
      - serviceAccountToken:
          audience: api
          expirationSeconds: 3600
          path: token
```

Der Beispielspod aht ein projected Volume, das den eingefügten ServiceAccount enthält. Dieses Token kann von einem Container des Pods verwendet werden um auf das Kubernetes API zuzugreifen. Das `audience` Feld enthält die beabsichtigte Audience des Tockens. Ein Empfänger des Tokens muss sich selbst mit dem Identifier, der in der Audience des Tokens angegeben ist, identifizieren und andernfalls das Token zurückweisen. Dieses Feld ist optional und ist standardmässig auf den Identifier des API Servers gesetzt.

Die `expirationSeconds` ist die erwartete Gültigkeitsdauer des ServiceAccount Tokens. Sie ist standardmässig auf 1 Stunde gesetzt und muss mindestens 10 Minuten (600 Sekunden) betragen. EinE AdministratorIn kann den Maximalwert durch Angabe der Option `--service-account-max-token-expiration` für den API Server begrenzen. Das Feld `path` gibt einen Pfad an, der relativ zum Mountpoint des projected Volume ist.

{{< note >}}
Ein Container, der ein projected Volume als [`subPath`](#using-subpath) verwendet, wird keine Änderungen an den Volume Quellen mitkriegen.
{{< /note >}}

### quobyte

Ein `quobyte` Volume erlaubt es ein existierendes [Quobyte](https://www.quobyte.com) Volume in einen Pod zu mounten.

{{< note >}}
Du musst dein eigenes Quobyte aufgesetzt und laufen haben und die Volumes erstellt bevor du es verwenden kannst.
{{< /note >}}

Quobyte unterstützt das {{< glossary_tooltip text="Container Storage Interface" term_id="csi" >}}.
CSI ist das empfohlene Plugin um Quobyte Volumes in Kubernetes zu nutzen. Quobyte's GitHub Projekt enthält [instructions](https://github.com/quobyte/quobyte-csi#quobyte-csi) zum Deployment von Quobyte mit CSI und auch Beispiele.

### rbd

Ein `rbd` Volume erlaubt es ein [Rados Block Device](https://ceph.com/docs/master/rbd/rbd/) (RBD) Volume in einen Pod zu mounten. Anders als `emptyDir`, das gelöscht wird, wenn ein Pod entfernt wird, bleibt der Inhalt eines `rbd` Volumes erhalten und das Volume wird lediglich unmounted.
into your Pod. Das bedeutet, dass ein RBD Volume vorab mit Daten gefüllt werden kann und dass Daten zwischen Pods geteilt werden können. 

{{< note >}}
Du musst eine Ceph Installation laufen haben bevor due RBD verwenden kannst.
You must have a Ceph installation running before you can use RBD.
{{< /note >}}

Ein Feature von RBD ist, dass es von mehreren Consumern gleichzeitig read-only gemounted werden kann. Das bedeutet, dass man ein Volume schon vorab mit einem Datenset bestücken kann und sie dann parallel an so viele Pods servieren wie nötig. Leider können RBD Volumes nur von einem einzelnen Consumer im read-write Modus gemounted werden. Gleichzeitige Schreibzugriffe sind nicht erlaubt.

Zu den Details vgl. [RBD example](https://github.com/kubernetes/examples/tree/{{< param "githubbranch" >}}/volumes/rbd).

### scaleIO (deprecated) {#scaleio}

ScaleIO ist eine software-basierte Storage Plattform, die existierende Hardware nutzt um Clusters von scalable shared block networked Storage zu erstellen. Das `scaleIO` Volume Plugin erlaubt es auf Pods existierende ScaleIO Volumes zuzugreifen. Für Information über dynamische Provisionierung von neuen Volumes für persistent Volume Claims vgl. [ScaleIO persistent volumes](/docs/concepts/storage/persistent-volumes/#scaleio).

{{< note >}}
Du musst einen existierenden ScaleIO Cluster aufgesetzt und laufen haben und die Volumes erstellt bevor du sie nutzen kannst.
{{< /note >}}

Das folgende Beispiel ist eine Pod-Konfiguration mit ScaleIO:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-0
spec:
  containers:
  - image: k8s.gcr.io/test-webserver
    name: pod-0
    volumeMounts:
    - mountPath: /test-pd
      name: vol-0
  volumes:
  - name: vol-0
    scaleIO:
      gateway: https://localhost:443/api
      system: scaleio
      protectionDomain: sd0
      storagePool: sp1
      volumeName: vol-0
      secretRef:
        name: sio-secret
      fsType: xfs
```

Zu den Details vgl. [ScaleIO](https://github.com/kubernetes/examples/tree/{{< param "githubbranch" >}}/staging/volumes/scaleio).

### secret

Ein `secret` Volume wird verwendet um sensitive Information, wie z.B. Passwörter, an Pods zu übergeben. Du kannst Secretes im Kubernetes API speichern und sie als Dateien für die Verwendung durch Pods mounten ohne direkte Koppelung an Kubernetes. `secrete` Volumes liegen auf einem tmpfs (einem Dateisystem im RAM) und werden so nie auf nicht-flüchtiges Storage geschreiben.

{{< note >}}
Du musst ein Secret im Kubernetes API erstellen bevor du es nutzen kannst.
{{< /note >}}

{{< note >}}
Ein Container, der ein Secret als [`subPath`](#using-subpath) Volume mounted wird Updates des Secrets nicht mitkriegen.
{{< /note >}}

Für mehr Details vgl. [Configuring Secrets](/docs/concepts/configuration/secret/).

### storageOS {#storageos}

Ein `storageos` Volume erlaubt es ein existierendes [StorageOS](https://www.storageos.com) Volume in einen Pod zu mounten.

StorageOS läuft als Container in deinem Kubernetes Environment und macht lokales oder eingehängtes Storage von jedem Node im Kubernetes Cluster aus zugänglich. Als Schutz gegen Node-Ausfälle k¨ønnen Daten repliziert werden. Thin provisioning und Kompression können die Kapazität verbessern und die Kosten reduzieren.

Im Grunde stellt StorageOS Containern Block Storage über ein Dateisystem zur Verfügung.

Der StorageOS Container verlangt 64-bit Linux und hat sonst keine Abhängigkeiten. Eine freie Entwicklerlizenz ist verfügbar.

{{< caution >}}
Du musst den StorageOS Container auf jedem Node, der auf StorageOS Volume zugreifen oder Storage zum Pool beetragen will. Für die Installationsanleitung vgl. [StorageOS documentation](https://docs.storageos.com).
{{< /caution >}}

Das folgende Beispiel ist eine Pod Konfiguration mit StorageOS:

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    name: redis
    role: master
  name: test-storageos-redis
spec:
  containers:
    - name: master
      image: kubernetes/redis:v1
      env:
        - name: MASTER
          value: "true"
      ports:
        - containerPort: 6379
      volumeMounts:
        - mountPath: /redis-master-data
          name: redis-data
  volumes:
    - name: redis-data
      storageos:
        # The `redis-vol01` volume must already exist within StorageOS in the `default` namespace.
        volumeName: redis-vol01
        fsType: ext4
```

Für mehr Information über StorageOS, dynamisches Provisionieren und PersistentVolumeClaims, vgl. die [StorageOS examples](https://github.com/kubernetes/examples/blob/master/volumes/storageos).

### vsphereVolume {#vspherevolume}

{{< note >}}
Du musst den Kubernetes vSphere Cloud Provider konfigurieren. Zur Cloud Provider Konfiguration vgl. den [vSphere Getting Started guide](https://vmware.github.io/vsphere-storage-for-kubernetes/documentation/).
{{< /note >}}

Ein `vsphereVolume` wird verwendet um ein vSphere VMDK Volume in einen Pod zu mounten. Der Inhalt des Volumes bleibt erhalten, wenn es unmounted wird. Es unterstützt sowohl VMFS als auch VSAN Datastores.

{{< note >}}
Du musst ein vSphere VMDK Volume mit einer der folgenden Methoden erstellen bevor du es in einem Pod nutzen kannst.
{{< /note >}}

#### Creating a VMDK volume {#creating-vmdk-volume}

Wähle eine der folgenden Methoden um ein VMDK zu erstellen.

{{< tabs name="tabs_volumes" >}}
{{% tab name="Create using vmkfstools" %}}
Logge dich zuerst per SSH auf dem ESX Host ein und verwende dann das folgende Kommando um ein VMDK zu erstellen:

```shell
vmkfstools -c 2G /vmfs/volumes/DatastoreName/volumes/myDisk.vmdk
```

{{% /tab %}}
{{% tab name="Create using vmware-vdiskmanager" %}}
Verwende das folgende Kommando um ein VMDK zu erstellen:

```shell
vmware-vdiskmanager -c -t 0 -s 40GB -a lsilogic myDisk.vmdk
```

{{% /tab %}}

{{< /tabs >}}

#### vSphere VMDK Beispielskonfiguration {#vsphere-vmdk-configuration}

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-vmdk
spec:
  containers:
  - image: k8s.gcr.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /test-vmdk
      name: test-volume
  volumes:
  - name: test-volume
    # This VMDK volume must already exist.
    vsphereVolume:
      volumePath: "[DatastoreName] volumes/myDisk"
      fsType: ext4
```

Für mehr Information vgl die [vSphere volume](https://github.com/kubernetes/examples/tree/master/staging/volumes/vsphere) Beispiele.

#### vSphere CSI migration {#vsphere-csi-migration}

{{< feature-state for_k8s_version="v1.19" state="beta" >}}

Das `CSIMigration` Feature von `vsphereVolume`, leitet, wenn es eingeschaltet ist, alle Plugin-Operationen vom existierenden in-tree Plugin zum `csi.vsphere.vmware.com` Container Storage Interface (CSI) Driver weiter. Um dieses Feature nutzen zu können, muss der [vSphere CSI driver](https://github.com/kubernetes-sigs/vsphere-csi-driver) installiert sein und die `CSIMigration` and `CSIMigrationvSphere`
[feature gates](/docs/reference/command-line-tools-reference/feature-gates/) müssen eingeschaltet sein.

Es verlangt auch, dass die Mindestversion von vSphere vCenter/ESXi 7.0u1 und die HW-Version mindestens 15 ist.

{{< note >}}
Die folgenden StorageClass Parameter aus dem eingebauten `vsphereVolume` Plugin werden vom vSphere CSI Driver nicht unterstützt:

* `diskformat`
* `hostfailurestotolerate`
* `forceprovisioning`
* `cachereservation`
* `diskstripes`
* `objectspacereservation`
* `iopslimit`

Existierende Volumes, die mit diesen Parametern erstellt wurden, werden auf den vSphere CSI Driver migriert, aber neue Volumes, die mit dem vSphere CSI Driver erstellt werden, werden diese Parameter nicht beachten.
{{< /note >}}

#### vSphere CSI migration complete {#vsphere-csi-migration-complete}

{{< feature-state for_k8s_version="v1.19" state="beta" >}}

Um das `vsphereVolume` Plugin daran zu hindern durch den Controller Manager und das Kubelet geladen zu werden, muss dieses Flag auf `true` gesetzt werden. Du muss einen `csi.vsphere.vmware.com` {{< glossary_tooltip text="CSI" term_id="csi" >}} Driver auf allen Worker Nodes installieren.

## Using subPath {#using-subpath}

Manchmal ist es nützlich ein einziges Volume für mehrere Zweck in einem einzigen Pod zu nutzne. Die Eigenschaft `volumeMounts.subPath` spezifiziert einen Pfad innerhalb des angegebenen Volumes anstelle seiner Wurzel.

Das folgende Beispiel zeigt wie man einen Pod mti einem LAMP Stack (Linux Apache MySQL PHP) konfigurert und nur ein einziges shared Volume verwendet. Diese beispielhafte `supPath` Konfiguration wird nicht für den produktiven Einsatz empfohlen.

Der Code und die Daten der PHP Applikation entsprechen dem `html` Folder des Volumes und die MySQL Datenbank ist im `mysql` Folder des Volumes abgelegt. Zum Beispiel:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-lamp-site
spec:
    containers:
    - name: mysql
      image: mysql
      env:
      - name: MYSQL_ROOT_PASSWORD
        value: "rootpasswd"
      volumeMounts:
      - mountPath: /var/lib/mysql
        name: site-data
        subPath: mysql
    - name: php
      image: php:7.0-apache
      volumeMounts:
      - mountPath: /var/www/html
        name: site-data
        subPath: html
    volumes:
    - name: site-data
      persistentVolumeClaim:
        claimName: my-lamp-site-data
```

### Verwendung von subPath mit expandierten Umgebungsvariablen {#using-subpath-expanded-environment}

{{< feature-state for_k8s_version="v1.17" state="stable" >}}

Verwende das `subPathExpr` Feld um `subPath` Verzeichnisnamen aus downward API Umgebungsvariablen zu erstellen. Die Properties `subPath` und `subPathExpr` schliessen einander gegenseitig aus.

In diesem Beispiel nutzt ein `Pod` eine `subPathExpr` um ein Verzeichnis `pod1` im `hostPath` Volume `/var/log/pods` zu erstellen. Das `hostPath` Volume nimmt den Namen des `Pods` vom `downwardAPI`. Das Verzeichnis `/var/log/pods/pod1` auf dem Host wird als `/logs/` in den Container gemounted.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod1
spec:
  containers:
  - name: container1
    env:
    - name: POD_NAME
      valueFrom:
        fieldRef:
          apiVersion: v1
          fieldPath: metadata.name
    image: busybox
    command: [ "sh", "-c", "while [ true ]; do echo 'Hello'; sleep 10; done | tee -a /logs/hello.txt" ]
    volumeMounts:
    - name: workdir1
      mountPath: /logs
      subPathExpr: $(POD_NAME)
  restartPolicy: Never
  volumes:
  - name: workdir1
    hostPath:
      path: /var/log/pods
```

## Resources

The storage media (such as Disk or SSD) of an `emptyDir` volume is determined by the
medium of the filesystem holding the kubelet root dir (typically
`/var/lib/kubelet`). There is no limit on how much space an `emptyDir` or
`hostPath` volume can consume, and no isolation between containers or between
pods.

To learn about requesting space using a resource specification, see
[how to manage resources](/docs/concepts/configuration/manage-resources-containers/).

## Out-of-tree volume plugins

The out-of-tree volume plugins include
{{< glossary_tooltip text="Container Storage Interface" term_id="csi" >}} (CSI)
and FlexVolume. These plugins enable storage vendors to create custom storage plugins
without adding their plugin source code to the Kubernetes repository.

Previously, all volume plugins were "in-tree". The "in-tree" plugins were built, linked, compiled,
and shipped with the core Kubernetes binaries. This meant that adding a new storage system to
Kubernetes (a volume plugin) required checking code into the core Kubernetes code repository.

Both CSI and FlexVolume allow volume plugins to be developed independent of
the Kubernetes code base, and deployed (installed) on Kubernetes clusters as
extensions.

For storage vendors looking to create an out-of-tree volume plugin, please refer
to the [volume plugin FAQ](https://github.com/kubernetes/community/blob/master/sig-storage/volume-plugin-faq.md).

### csi

[Container Storage Interface](https://github.com/container-storage-interface/spec/blob/master/spec.md)
(CSI) defines a standard interface for container orchestration systems (like
Kubernetes) to expose arbitrary storage systems to their container workloads.

Please read the [CSI design proposal](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/storage/container-storage-interface.md) for more information.

{{< note >}}
Support for CSI spec versions 0.2 and 0.3 are deprecated in Kubernetes
v1.13 and will be removed in a future release.
{{< /note >}}

{{< note >}}
CSI drivers may not be compatible across all Kubernetes releases.
Please check the specific CSI driver's documentation for supported
deployments steps for each Kubernetes release and a compatibility matrix.
{{< /note >}}

Once a CSI compatible volume driver is deployed on a Kubernetes cluster, users
may use the `csi` volume type to attach or mount the volumes exposed by the
CSI driver.

A `csi` volume can be used in a Pod in three different ways:

* through a reference to a [PersistentVolumeClaim](#persistentvolumeclaim)
* with a [generic ephemeral volume](/docs/concepts/storage/ephemeral-volumes/#generic-ephemeral-volume)
(alpha feature)
* with a [CSI ephemeral volume](/docs/concepts/storage/ephemeral-volumes/#csi-ephemeral-volume)
if the driver supports that (beta feature)

The following fields are available to storage administrators to configure a CSI
persistent volume:

* `driver`: A string value that specifies the name of the volume driver to use.
  This value must correspond to the value returned in the `GetPluginInfoResponse`
  by the CSI driver as defined in the [CSI spec](https://github.com/container-storage-interface/spec/blob/master/spec.md#getplugininfo).
  It is used by Kubernetes to identify which CSI driver to call out to, and by
  CSI driver components to identify which PV objects belong to the CSI driver.
* `volumeHandle`: A string value that uniquely identifies the volume. This value
  must correspond to the value returned in the `volume.id` field of the
  `CreateVolumeResponse` by the CSI driver as defined in the [CSI spec](https://github.com/container-storage-interface/spec/blob/master/spec.md#createvolume).
  The value is passed as `volume_id` on all calls to the CSI volume driver when
  referencing the volume.
* `readOnly`: An optional boolean value indicating whether the volume is to be
  "ControllerPublished" (attached) as read only. Default is false. This value is
  passed to the CSI driver via the `readonly` field in the
  `ControllerPublishVolumeRequest`.
* `fsType`: If the PV's `VolumeMode` is `Filesystem` then this field may be used
  to specify the filesystem that should be used to mount the volume. If the
  volume has not been formatted and formatting is supported, this value will be
  used to format the volume.
  This value is passed to the CSI driver via the `VolumeCapability` field of
  `ControllerPublishVolumeRequest`, `NodeStageVolumeRequest`, and
  `NodePublishVolumeRequest`.
* `volumeAttributes`: A map of string to string that specifies static properties
  of a volume. This map must correspond to the map returned in the
  `volume.attributes` field of the `CreateVolumeResponse` by the CSI driver as
  defined in the [CSI spec](https://github.com/container-storage-interface/spec/blob/master/spec.md#createvolume).
  The map is passed to the CSI driver via the `volume_context` field in the
  `ControllerPublishVolumeRequest`, `NodeStageVolumeRequest`, and
  `NodePublishVolumeRequest`.
* `controllerPublishSecretRef`: A reference to the secret object containing
  sensitive information to pass to the CSI driver to complete the CSI
  `ControllerPublishVolume` and `ControllerUnpublishVolume` calls. This field is
  optional, and may be empty if no secret is required. If the Secret
  contains more than one secret, all secrets are passed.
* `nodeStageSecretRef`: A reference to the secret object containing
  sensitive information to pass to the CSI driver to complete the CSI
  `NodeStageVolume` call. This field is optional, and may be empty if no secret
  is required. If the Secret contains more than one secret, all secrets
  are passed.
* `nodePublishSecretRef`: A reference to the secret object containing
  sensitive information to pass to the CSI driver to complete the CSI
  `NodePublishVolume` call. This field is optional, and may be empty if no
  secret is required. If the secret object contains more than one secret, all
  secrets are passed.

#### CSI raw block volume support

{{< feature-state for_k8s_version="v1.18" state="stable" >}}

Vendors with external CSI drivers can implement raw block volume support
in Kubernetes workloads.

You can set up your
[PersistentVolume/PersistentVolumeClaim with raw block volume support](/docs/concepts/storage/persistent-volumes/#raw-block-volume-support) as usual, without any CSI specific changes.

#### CSI ephemeral volumes

{{< feature-state for_k8s_version="v1.16" state="beta" >}}

You can directly configure CSI volumes within the Pod
specification. Volumes specified in this way are ephemeral and do not
persist across pod restarts. See [Ephemeral
Volumes](/docs/concepts/storage/ephemeral-volumes/#csi-ephemeral-volume)
for more information.

For more information on how to develop a CSI driver, refer to the
[kubernetes-csi documentation](https://kubernetes-csi.github.io/docs/)

#### Migrating to CSI drivers from in-tree plugins

{{< feature-state for_k8s_version="v1.17" state="beta" >}}

The `CSIMigration` feature, when enabled, directs operations against existing in-tree
plugins to corresponding CSI plugins (which are expected to be installed and configured).
As a result, operators do not have to make any
configuration changes to existing Storage Classes, PersistentVolumes or PersistentVolumeClaims
(referring to in-tree plugins) when transitioning to a CSI driver that supersedes an in-tree plugin.

The operations and features that are supported include:
provisioning/delete, attach/detach, mount/unmount and resizing of volumes.

In-tree plugins that support `CSIMigration` and have a corresponding CSI driver implemented
are listed in [Types of Volumes](#volume-types).

### flexVolume

FlexVolume is an out-of-tree plugin interface that has existed in Kubernetes
since version 1.2 (before CSI). It uses an exec-based model to interface with
drivers. The FlexVolume driver binaries must be installed in a pre-defined volume
plugin path on each node and in some cases the control plane nodes as well.

Pods interact with FlexVolume drivers through the `flexvolume` in-tree volume plugin.
For more details, see the [FlexVolume](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-storage/flexvolume.md) examples.

## Mount propagation

Mount propagation allows for sharing volumes mounted by a container to
other containers in the same pod, or even to other pods on the same node.

Mount propagation of a volume is controlled by the `mountPropagation` field
in `Container.volumeMounts`. Its values are:

* `None` - This volume mount will not receive any subsequent mounts
  that are mounted to this volume or any of its subdirectories by the host.
  In similar fashion, no mounts created by the container will be visible on
  the host. This is the default mode.

  This mode is equal to `private` mount propagation as described in the
  [Linux kernel documentation](https://www.kernel.org/doc/Documentation/filesystems/sharedsubtree.txt)

* `HostToContainer` - This volume mount will receive all subsequent mounts
  that are mounted to this volume or any of its subdirectories.

  In other words, if the host mounts anything inside the volume mount, the
  container will see it mounted there.

  Similarly, if any Pod with `Bidirectional` mount propagation to the same
  volume mounts anything there, the container with `HostToContainer` mount
  propagation will see it.

  This mode is equal to `rslave` mount propagation as described in the
  [Linux kernel documentation](https://www.kernel.org/doc/Documentation/filesystems/sharedsubtree.txt)

* `Bidirectional` - This volume mount behaves the same the `HostToContainer` mount.
  In addition, all volume mounts created by the container will be propagated
  back to the host and to all containers of all pods that use the same volume.

  A typical use case for this mode is a Pod with a FlexVolume or CSI driver or
  a Pod that needs to mount something on the host using a `hostPath` volume.

  This mode is equal to `rshared` mount propagation as described in the
  [Linux kernel documentation](https://www.kernel.org/doc/Documentation/filesystems/sharedsubtree.txt)

  {{< warning >}}
  `Bidirectional` mount propagation can be dangerous. It can damage
  the host operating system and therefore it is allowed only in privileged
  containers. Familiarity with Linux kernel behavior is strongly recommended.
  In addition, any volume mounts created by containers in pods must be destroyed
  (unmounted) by the containers on termination.
  {{< /warning >}}

### Configuration

Before mount propagation can work properly on some deployments (CoreOS,
RedHat/Centos, Ubuntu) mount share must be configured correctly in
Docker as shown below.

Edit your Docker's `systemd` service file. Set `MountFlags` as follows:

```shell
MountFlags=shared
```

Or, remove `MountFlags=slave` if present. Then restart the Docker daemon:

```shell
sudo systemctl daemon-reload
sudo systemctl restart docker
```

## {{% heading "whatsnext" %}}

Follow an example of [deploying WordPress and MySQL with Persistent Volumes](/docs/tutorials/stateful-application/mysql-wordpress-persistent-volume/).
