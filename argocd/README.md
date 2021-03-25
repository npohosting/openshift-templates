# argocd-install

In deze Readme leg ik uit hoe een ArgoCD Cluster kan worden geinstalleerd op het NPO CHP4 platform. 

## Vereisten:

* oc client geinstalleerd
* argocd client geinstalleerd: https://argoproj.github.io/argo-cd/cli_installation/

## Stap 1: Aanmaken Project

Het ArgoCD cluster moet worden geinstalleerd in een eigen Project. Maak die eerst zelf aan. In dit voorbeeld gebruiken we het project `hens-argocd-demo`:

```bash
oc new-project hens-argocd
```

Wanneer het project is aangemaakt kan een aanvraag ingediend worden via https://support.npohosting.nl voor het installeren van de ArgoCD operator. Pas wanneer deze is afgehandeld kunnen we verder naar stap 2.

## Stap 2: Installeren ArgoCD

1. Zorg voor een lokale checkout van deze repository, en wandel de `openshift-templates/argocd` directory in:

```bash
git clone https://github.com/npohosting/openshift-templates.git
cd openshift-templates/argocd
```

2. In de directory `openshift-templates/argocd` kunnen we het `argocd-install` commando aanroepen. Als we dat met `-h` doen kunnen we de diverse opties zien:

```bash
./argocd-install -h
Usage: argocd-install [options <arguments>]
    -n      Name of the ArgoCD Cluster.
    -p      Namespace where to install the ArgoCD Cluster.
    -c      Which organization are you (valid options are: nos, hens).
    -t      Which Namespace needs to be managed by ArgoCD.
    -H      Hostname of the ArgoCD Cluster.
    -a      This option allows you to give ArgoCD access to a new project, without installing an entire cluster.
```

Hieronder staan de opties nader uitgelegd:

* `-n`: Dit wordt de naam van het ArgoCD cluster, en dus ook de naam van alle objecten die er worden gemaakt.
* `-p`: Dit is de naam van het project waar het ArgoCD cluster moet komen te draaien. Dit is dus het project waar de ArgoCD operator in is geinstalleerd.
* `-c`: Hiermee geef je aan voor welke omroep dit is. Dit staat gelijk aan de prefix die normaal gesproken voor de naam van een project wordt gezet.
* `-t`: Optioneel. Hiermee kan je een project opgeven waar ArgoCD in moet kunnen werken. Wanneer het project niet bestaat, wordt het aangemaakt door het script.
* `-H`: Optioneel. Hiermee bepaal je onder welke hostname het ArgoCD cluster beschikbaar moet komen. Als je dit niet opgeeft, dan gaat het script zelf een suggestie geven. In beide gevallen worden de routes voorzien van een Letsencrypt certificaat.
* `-a`: Met deze optie kan je ArgoCD toegang geven tot andere projecten na de installatie. 

3. Als we weten welke waarden we willen meegeven aan de installatie, dan kunnen we hem starten:
```bash
./argocd-install -p hens-argocd-demo -n argocd-demo -c hens -t hens-sample-app -H argocd.apps.hens.cluster.chp4.io
argocd.argoproj.io/argocd-demo created
route.route.openshift.io/argocd-demo created
Waiting for ArgoCD Cluster to become available.
Waiting for ArgoCD Cluster to become available.
Waiting for ArgoCD Cluster to become available.
...<dit kan even duren>...
Waiting for ArgoCD Cluster to become available.
Waiting for ArgoCD Cluster to become available.
Waiting for ArgoCD Cluster to become available.
appproject.argoproj.io/default configured
rolebinding.rbac.authorization.k8s.io/argocd-edit-role created
```

4. Nu is het cluster klaar voor gebruik. Dit kan voor de zekerheid nog even gecontroleerd worden middels een `oc get pods`:

```bash
oc get pods
NAME                                                 READY   STATUS    RESTARTS   AGE
argocd-demo-application-controller-c8fd54f74-gvk27   1/1     Running   0          72s
argocd-demo-dex-server-66cd66bb9-nl6gh               1/1     Running   0          72s
argocd-demo-redis-778bb79fbd-qjxq2                   1/1     Running   0          72s
argocd-demo-repo-server-77d4c544cf-vn65p             1/1     Running   0          72s
argocd-demo-server-7b656f8f78-wt86m                  1/1     Running   0          72s
argocd-operator-d976b9d6f-j9d45                      1/1     Running   0          107s
```

## Stap 3: Eerste keer inloggen

Nu kunnen we voor het eerst inloggen. ArgoCD maakt bij de installatie een `admin` user aan. Het wachtwoord is te vinden in een secret in de ArgoCD namespace genaamd: `<naam-van-je-cluster>-cluster`. In dit voorbeeld dus `argocd-demo-cluster`.

```bash
oc get secret argocd-demo-cluster
NAME                  TYPE     DATA   AGE
argocd-demo-cluster   Opaque   1      4m6s
```

Als we naar de yaml van dat secret kijken, dan vinden we een veld `admin.password`:

```bash
oc get secret argocd-demo-cluster -o yaml
```

```yaml
apiVersion: v1
data:
  admin.password: <heel geheim>
kind: Secret
metadata:
...
```

De waarde van `admin.password` kunnen we decoderen met `base64`:

```bash
echo -n '<heel geheim>' | base64 -d
<nog geheimer>
```

De output die je nu hebt gekregen is het admin wachtwoord waarmee je kan inloggen op je ArgoCD cluster via zowel de webinterface als de CLI.

## Stap 4: Aanmaken extra user accounts

Het aanmaken van extra accounts doen we, deels, met de ArgoCD CLI en bestaat uit de volgende stappen:

1. Aanmaken van het account

Accounts worden aangemaakt via de `argocd-cm` ConfigMap:

```bash
oc edit configmap argocd-cm -n hens-argocd-demo
```

Voeg in de ConfigMap onder `data:` toe `accounts.<username>: apiKey, login`. Vervang <username> door een username, bijvoorbeeld "tim":

```yaml
...
apiVersion: v1
data:
  accounts.tim: apiKey, login
  application.instanceLabelKey: ""
  configManagementPlugins: ""
  dex.config: ""
  ga.anonymizeusers: "false"
  ga.trackingid: ""
... 
```

Log nu in met de admin account op het ArgoCD cluster via de CLI:

```bash
argocd login --grpc-web=true argocd-test.apps.hens.cluster.chp4.io
Username: admin
Password:
'admin' logged in successfully
Context 'argocd.apps.hens.cluster.chp4.io' updated
```

Je kan nu controleren of het account is aangemaakt, door alle accounts op te vragen:
```bash
argocd account list                                                                                                                                                            
NAME   ENABLED  CAPABILITIES
admin  true     login
tim    true     apiKey, login
```

2. Installen van het wachtwoord

Het instellen van het wachtwoord doen we via de CLI. We moeten hiervoor zijn ingelogd met een account met admin rechten. Als je het commando aanroept vraagt hij op een `current password`. Dit is het wachtwoord van het account waarmee je op dat moment bent ingelogd.

```bash
argocd account update-password --account tim
*** Enter current password: (Het wachtwoord van het account waarmee nu is ingelogd (in dit voorbeeld **admin**)
*** Enter new password: (Het nieuwe wachtwoord voor user **tim**)
*** Confirm new password: (Herhaal het nieuwe wachtwoord)
Password updated
```

3. Toekennen van rechten

Na het aanmaken van het account en het instellen van het wachtwoord, moet je (mogelijk) nog rechten toekennen aan het account. ArgoCD kent standaard twee rollen: `readonly` of `admin`, default wordt een nieuwe gebruiker aangemaakt met `readonly` rechten.

Rechten kunnen we instellen in de `argocd-rbac-cm`. Daar vind je een veld met de naam `policy.csv`. Bij een nieuwe installatie is die leeg. Aan de `policy.csv` voegen we `g, tim, role:admin` toe.

```yaml
...
apiVersion: v1
data:
  policy.csv: |
    g, tim, role:admin
  policy.default: role:readonly
  scopes: '[groups]'
kind: ConfigMap
...
```

Nu kan je inloggen met de user `tim`.

## Stap 5: Uitschakelen admin account

Als je een useraccount met admin rechten hebt gemaakt, dan wil je het admin account uitschakelen. Doe dit door de regel `admin.enabled: "false` toe te voegen aan de `argocd-cm` configmap:

```bash
oc edit configmap argocd-cm
```

```yaml
...
data:
  admin.enabled: "false"
...
```

## Stap 6: ArgoCD toegang geven tot extra projecten na de installatie.

Met dit script kan je ook ArgoCD toegang geven tot andere projecten na de installatie. Dit kan middels de `-a` vlag:

```bash
./argocd-install -p hens-argocd-demo -n argocd-demo -t hens-sample-app -a
```

Op dit moment ondersteund deze optie maar 1 project tegelijk, dus dit moet je voor elk project herhalen.

## Stap 7: Klaar.

Je ArgoCD installatie is nu klaar voor gebruik, onderstaande nog een paar handige links:

* ArgoCD Documentatie: https://argoproj.github.io/argo-cd/
* Documentatie over de interne RBAC: https://argoproj.github.io/argo-cd/operator-manual/rbac/
* Documentatie over usermanagement en overige authenticatie methodes: https://argoproj.github.io/argo-cd/operator-manual/user-management/