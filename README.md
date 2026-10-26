# LAB-2490: Stan's Robot Shop on OpenShift with IBM Instana, Turbonomic and Cloudability

Hands-on lab where you deploy Stan's Robot Shop (a polyglot microservices demo app) to a shared Red Hat OpenShift 4.x cluster on IBM TechZone, generate load, and then observe, optimise and cost-manage it.

| Phase | What you do | Product |
|---|---|---|
| 1 | Deploy Robot Shop to OpenShift with Helm | OpenShift / Helm |
| 2 | Generate synthetic load | Load generator |
| 3 | Deploy the agent and observe the app | IBM Instana |
| 4 | Review resource optimisation recommendations | IBM Turbonomic |
| 5 | Review cloud and container costs | IBM Cloudability |

## Lab materials

- [`2490-lab-guide.docx`](2490-lab-guide.docx): step-by-step lab guide (follow this)
- [`Lab-2490-Deck.pdf`](Lab-2490-Deck.pdf): lab presentation deck

## Prerequisites

- `oc` CLI (OpenShift CLI v4.x)
- `helm` CLI (v3.x)
- `git`
- Access to the TechZone OpenShift cluster (credentials are in the lab guide)

### Windows setup

Use **PowerShell** (run as a normal user). Install the three tools, then open a **new** PowerShell window so the updated PATH is picked up.

| Tool | Download | Or install with winget |
|---|---|---|
| `git` | [git-scm.com/download/win](https://git-scm.com/download/win) | `winget install --id Git.Git -e` |
| `helm` | [helm.sh/docs/intro/install](https://helm.sh/docs/intro/install/) | `winget install --id Helm.Helm -e` |
| `oc` | [openshift-client-windows.zip](https://mirror.openshift.com/pub/openshift-v4/clients/ocp/stable/openshift-client-windows.zip) | Not available, use the zip |

Installing `oc`:

1. Download the zip above and extract it (for example to `C:\oc`). It contains `oc.exe`.
2. Add that folder to your PATH: Start menu > "Edit the system environment variables" > Environment Variables > select `Path` under your user > Edit > New > `C:\oc`.
3. Open a new PowerShell window.

Check the install:

```powershell
git --version
helm version
oc version --client
```

Windows differences in the steps below:

- Replace `export VAR=value` with `$env:VAR="value"` in PowerShell.
- `envsubst` is not available on Windows. Use the PowerShell snippet in [Instana agent (Phase 3)](#instana-agent-phase-3) instead.
- The `oc`, `helm` and `git` commands in the quick start run unchanged in PowerShell.

## Quick start

30 students share the cluster, so pick a short, lowercase unique ID (2-8 characters, e.g. `ab07`) and use it wherever you see `<YOUR-ID>`.

```shell
# 1. Log in (token and server URL are in the lab guide)
oc login --token=<TOKEN> --server=<API-SERVER-URL>

# 2. Clone this repository
git clone https://github.com/AishaDarga/LAB-2490.git
cd LAB-2490

# 3. Create your namespace and grant the required SCCs
oc new-project robot-shop-<YOUR-ID>
oc adm policy add-scc-to-user anyuid -z default -n robot-shop-<YOUR-ID>
oc adm policy add-scc-to-user privileged -z default -n robot-shop-<YOUR-ID>
oc adm policy add-scc-to-user anyuid -z robot-shop -n robot-shop-<YOUR-ID>

# 4. Deploy Robot Shop
helm install robot-shop K8s/helm --set openshift=true --set ocCreateRoute=true -n robot-shop-<YOUR-ID>

# 5. Deploy the load generator
oc -n robot-shop-<YOUR-ID> apply -f K8s/load-deployment.yaml
```

The application is then available at `https://web-robot-shop-<YOUR-ID>.apps.<cluster-domain>`.

## Instana agent (Phase 3)

Create the agent custom resource from the template, supplying your own values (the agent key is in the Instana UI or the lab guide):

```shell
export INSTANA_ZONE=<zone>
export INSTANA_CLUSTER_NAME=<cluster>
export INSTANA_AGENT_KEY=<agent key>
export INSTANA_ENDPOINT_HOST=<e.g. ingress-orange-saas.instana.io>

oc new-project instana-agent
oc apply -f https://github.com/instana/instana-agent-operator/releases/latest/download/instana-agent-operator.yaml
envsubst < instana-agent.customresource.yaml.template | oc apply -n instana-agent -f -
```

On Windows (PowerShell):

```powershell
$env:INSTANA_ZONE="<zone>"
$env:INSTANA_CLUSTER_NAME="<cluster>"
$env:INSTANA_AGENT_KEY="<agent key>"
$env:INSTANA_ENDPOINT_HOST="<e.g. ingress-orange-saas.instana.io>"

oc new-project instana-agent
oc apply -f https://github.com/instana/instana-agent-operator/releases/latest/download/instana-agent-operator.yaml

$t = Get-Content instana-agent.customresource.yaml.template -Raw
foreach ($v in 'INSTANA_ZONE','INSTANA_CLUSTER_NAME','INSTANA_AGENT_KEY','INSTANA_ENDPOINT_HOST') {
  $t = $t.Replace('${' + $v + '}', (Get-Item "env:$v").Value)
}
$t | oc apply -n instana-agent -f -
```

