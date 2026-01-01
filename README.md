# namespace-factory-proj
## Starting out
After using minikube and starting argo using the following commands I started planning out the deployment strategy and the workflow of creating a namespace
```bash
k create namespace argocd
k apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
## Repository Directory Structure
following the yaml that was provided I started planning the directory structure
Tenant specification (example)
```yaml
namespace: team-a-dev
owner: team-a
environment: dev
quota:
cpuRequests: '4'
cpuLimits: '8'
memoryRequests: 8Gi
memoryLimits: 16Gi
limits:
defaultCpu: 500m
defaultMemory: 512Mi
defaultRequestCpu: 200m
defaultRequestMemory: 256Mi
rbac:
adminGroup: team-a-admins
viewGroup: team-a-viewers
```

The first thing I noticed is that if we apply the current yaml is that it enables our repository will to ignore standardization and look like the following example:
```
team.yaml
team-project.yaml
project.yaml
```

So in order to enforce standardization I decided to start by adding environment and team directories, and project files inside those that argocd will automatically use to define the namespace, owner, and environment, making our directory look like the following example:
```
envs/
├── dev
│   └── alpha
│       └── collector-project.yaml
└── prd
    └── alpha
        ├── genesis.yaml
        ├── test.yaml
        └── random-name.yaml
```

Now that we chose a new directory structure and a high-level workflow of creating a new namespace (creating a dir for the team -> creating a file for the project)
now whats left for us is to define how each project.yaml should look like

First I chose to replace the namespace name from team-a-dev to the convention [team]-[project]-[env], that is because we want to be able to filter our projects firstly by the team and then by the project name
its also important to note that the env variable is last for 2 reasons:
	- when inside the cluster we dont want to write the env name every time when we filter projects since we know where we are.
	- when our monitoring solution will gather data the namespace section will have the environment making it easier for us to know where the data was gathered from.

since we cant get our team's name, our env name and our project's name we can get the following variables without writing them in the project value files in our directory:
- namespace
- owner
- environment

now our value.yaml will be:
```yaml
quota:
	cpuRequests: '4'
	cpuLimits: '8'
	memoryRequests: 8Gi
	memoryLimits: 16Gi
limits:
	defaultCpu: 500m
	defaultMemory: 512Mi
	defaultRequestCpu: 200m
	defaultRequestMemory: 256Mi
rbac:
	adminGroup: team-a-admins
	viewGroup: team-a-viewers
```
We can now reorganize the file, and after doing so we will be left with:
```yaml
rbac:
  groups:
    admin: team-a-admins
    viewer: team-a-viewers

quota:
  cpu:
    requests: '4'
    limits: '8'
  memory:
    requests: "8Gi"
    limits: "16Gi"

limits:
  cpu:
    default: "500m"
    request: "200m"
  memory:
    default: "512Mi"
    request: "256Mi"
```
Even though the new file is a bit longer, its a bit easier on the eyes enabling us to get as much information as possible in one glance

Now that we finished planning the workflow and the file structure we can move on to creating the aplicationSets and helm charts by thinking of the factory steps when running

## ApplicationSets and Helm Charts
To create a new namespace following our standards from start to finish we will need a few things
```yaml
- app-project (for argocd)
- namespace.yaml
- networkpolicy.yaml
- limit-range.yaml
- resource-quota.yaml
- serviceAccount.yaml
- rbac-admin.yaml
- rbac-viewer.yaml
```

Since each of these files will need to have sections of it replaced a good tool for the job will be either kustomize or helm, and since helm can handle these values files in a easier manner than kustomize (which will require more text to do the same job), we can decide to start configuring these files to fit a helm chart.

Since we will need to create both our app-projects and applications and want to achieve security from tenants messing with out cluster we will want to have 2 applicationSets
- app-projects applicationSet
- team-applications applicationSet

This is done to
- Solve dependencies - since every application must belong to an appProject in argocd, creating an app-project applicationSet and configuring it to sync before our team-applications we ensure every run will have an appProject before the application itself
- Restrictions for tenants - since our applicationSets are separated we can run place the tenant's applicationSet to run within a more restricted project than default (which is a high-privilege project)
- Blast radius reduction - by separating the applicationSets we can ensure that if our team-applications applicationSet breaks for any reason our appProject applicationSet remains intact (which is also changed less frequently then the former)

now we can start creating the applicationSets and the Charts with that in mind.

Following our logic we can create our helm charts like so:
```
charts/
├── project-template
│   ├── Chart.yaml
│   └── templates
│       └── app-project.yaml
└── tenant-namespace
    ├── Chart.yaml
    ├── templates
    │   ├── limit-range.yaml
    │   ├── namespace.yaml
    │   ├── networkpolicy.yaml
    │   ├── rbac-admin.yaml
    │   ├── rbac-viewer.yaml
    │   ├── resource-quota.yaml
    │   └── serviceAccount.yaml
    └── values.yaml
```

The project-template chart is our app-project for argocd and will contain the app-project.yaml
and the tenant-namespace chart will be the one to contain our cluster yamls

After creating the Charts we can move on to create the following applicationSets
```
root-app.yaml               # The root application of our argocd instance
team-app-projects.yaml      # The appProjects applicationSet
team-applications.yaml      # The tenant's namespace related yamls applicationSet
```

### root application
our root app will contain our repository and is pretty standard and will include only our team-\*.yaml files since these are the applicationSets we will use in this exercise
its important to note that SyncPolicy that was added is there to ensure that we are always in our desired state according to our SSOT.
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-management-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/omri2003/namespace-factory-proj.git'
    targetRevision: HEAD
    path: .
    directory:
      include: 'team-*.yaml'
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - PruneLast=true
```

### team-app-projects
This application is configured to use the naming convention [team]-[env]-project-deployer in order to create applications related to the appProjects
Its purpose is to scan the folders under the envs and pass the "env" and "team" parameters in order to create an application that will use our appProject chart and create an appProject for our tenants.
We can also see that our sync-wave is set to "-20" which will ensure our appProject will be created before any of the resources from the  tenant related applicationSet (which we will soon see is set to "-10")
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: team-app-projects
  namespace: argocd
spec:
  goTemplate: true
  generators:
    - git:
        repoURL: 'https://github.com/omri2003/namespace-factory-proj.git'
        revision: main
        directories:
          - path: envs/*/*
  template:
    metadata:
      # Naming convention: [team]-[env]-project-deployer
      name: '{{index .path.segments 2}}-{{index .path.segments 1}}-project-deployer'
      annotations:
        argocd.argoproj.io/sync-wave: "-20"
    spec:
      project: default
      source:
        repoURL: 'https://github.com/omri2003/namespace-factory-proj.git'
        targetRevision: main
        path: charts/project-template
        helm:
          parameters:
            - name: "env"
              value: '{{index .path.segments 1}}'
            - name: "team"
              value: '{{index .path.segments 2}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: argocd
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```


### team-applications
This applicationSet's purpose is to discover each of our tenant's projects by scanning the envs directory, then generate their namespaces following the naming "team-project-env" and apply all of the resources under our tenant-namespace chart using the "env", "team", and "project" variables.
We can also see that the sync-wave here is set to -10 ensuring our appProject will be created before the applicationSet
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: team-applications
  namespace: argocd
spec:
  goTemplate: true
  generators:
    - git:
        repoURL: 'https://github.com/omri2003/namespace-factory-proj.git'
        revision: main
        requeueAfterSeconds: 60
        files:
          - path: "envs/*/*/*.yaml"
  template:
    metadata:
      # Naming: alpha-myproj-dev
      # We use 'index' safely here because the generator path filter (envs/*/*/*.yaml)
      # should only return files at that depth.
      name: '{{index .path.segments 2}}-{{.path.filename | trimSuffix ".yaml"}}-{{index .path.segments 1}}'
      annotations:
        argocd.argoproj.io/sync-wave: "-10"
    spec:
      # Naming convetion: project-[team]-[env]
      project: 'project-{{index .path.segments 2}}-{{index .path.segments 1}}'
      source:
        repoURL: 'https://github.com/omri2003/namespace-factory-proj.git'
        targetRevision: main
        path: charts/tenant-namespace
        helm:
          valueFiles:
            - '/{{.path.path}}/{{.path.filename}}'
          parameters:
            - name: "env"
              value: '{{index .path.segments 1}}'
            - name: "team"
              value: '{{index .path.segments 2}}'
            - name: "project"
              value: '{{.path.filename | trimSuffix ".yaml"}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{index .path.segments 2}}-{{.path.filename | trimSuffix ".yaml"}}-{{index .path.segments 1}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

## Charts
### project-template
This template relates to our appProject and ensures that
- we restrict our client by defining clusterResourcesWhitelist with only the Namespace in it thsu restricting the AppProject
- we are defining a destination that only follows the "team-.\*-env" convention thus restricting the tenant to only those namespaces

The only downside is sourceRepos which I have chosen to keep as a wildcard since those are configured based on the environment we are in and our regulations, which would be our organization's url.
```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: project-{{ .Values.team }}-{{ .Values.env }}
  namespace: argocd
spec:
  description: "Governance for Team {{ .Values.team }} in {{ .Values.env }}"
  sourceRepos: ["*"]
  destinations:
  - namespace: '{{ .Values.team }}-*-{{ .Values.env }}'
    server: https://kubernetes.default.svc
  clusterResourceWhitelist:
  - group: ''
    kind: Namespace
  namespaceResourceWhitelist:
  - group: '*'
    kind: '*'
```
### tenant-namespace
is this chart we will create each of our cluster yamls with the following sync order:
- -5 Namespace
- -4 Network Policy
- -3 Resource Quotas & Limit Range
- -2 Service Account
- -1 RBAC resources
this sync order was chosen based on the idea of first creating the environment, then applying its network and resource limits, creating the service account, and only after all of the limits and resources have been created allow the tenant to start using the project.

Its important to note that each file here will have the following labels:
```yaml
  labels:
    team: {{ .Values.team | quote }}
    project: {{ .Values.project | quote }}
    env: {{ .Values.env | quote }}
    managed-by: "namespace-factory"
```
these are added in order to make both monitoring and filtering easier for us and for the general ordering of the cluster (the more teams we have the more values those will hold).
#### namespace.yaml
Creates our namespace even though the CreateNamespace=true is enabled in our applicationSet, since we want to add our own metadata to the file.
also its important to have all files in our SSOT to ensure our desired state is always available.
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: {{ .Values.team }}-{{ .Values.project }}-{{ .Values.env }}
  labels:
    team: {{ .Values.team | quote }}
    project: {{ .Values.project | quote }}
    env: {{ .Values.env | quote }}
    managed-by: "namespace-factory"
  annotations:
    argocd.argoproj.io/sync-wave: "-5"
```
#### networkpolicy.yaml
Here we define a network policy that will only allow pods in the same namespace to "talk" to each other to make sure no pods from namespace-a are "talking" to namespace-b without us knowing.
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-only-same-namespace
  namespace: {{ .Values.team }}-{{ .Values.project }}-{{ .Values.env }}
  labels:
    team: {{ .Values.team | quote }}
    project: {{ .Values.project | quote }}
    env: {{ .Values.env | quote }}
    managed-by: "namespace-factory"
  annotations:
    argocd.argoproj.io/sync-wave: "-4"
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector: {}
```
#### resource-quota.yaml 
This Resource quota allows us to limit the namespace thus preventing noisy neighbors from affecting other tenants in the cluster.
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: {{ .Values.project }}-quota
  namespace: {{ .Values.team }}-{{ .Values.project }}-{{ .Values.env }}
  labels:
    team: {{ .Values.team | quote }}
    project: {{ .Values.project | quote }}
    env: {{ .Values.env | quote }}
    managed-by: "namespace-factory"
  annotations:
    argocd.argoproj.io/sync-wave: "-3"
spec:
  hard:
    requests.cpu: {{ .Values.quota.cpu.requests | quote }}
    limits.cpu: {{ .Values.quota.cpu.limits | quote }}
    requests.memory: {{ .Values.quota.memory.requests | quote }}
    limits.memory: {{ .Values.quota.memory.limits | quote }}
```
#### limit-range.yaml
This limit range will enforce limitations on the containers created thus preventing any pod from consuming too much of our resources, ensuring it is OOM killed in case it crosses the limitations set.
We also provide defaults that our developers can rely on when deploying pods without a resources section.
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: {{ .Values.project }}-limits
  namespace: {{ .Values.team }}-{{ .Values.project }}-{{ .Values.env }}
  labels:
    team: {{ .Values.team | quote }}
    project: {{ .Values.project | quote }}
    env: {{ .Values.env | quote }}
    managed-by: "namespace-factory"
  annotations:
    argocd.argoproj.io/sync-wave: "-3"
spec:
  limits:
  - type: Container
    default:
      cpu: {{ .Values.limits.cpu.default | quote }}
      memory: {{ .Values.limits.memory.default | quote }}
    defaultRequest:
      cpu: {{ .Values.limits.cpu.request | quote }}
      memory: {{ .Values.limits.memory.request | quote }}
```
#### serviceAccount.yaml
We create the following service account for cicd purposes, ensuring one will be created for each project the team owns.
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ .Values.project }}-pipeline-deployer
  namespace: {{ .Values.team }}-{{ .Values.project }}-{{ .Values.env }}
  labels:
    team: {{ .Values.team | quote }}
    project: {{ .Values.project | quote }}
    env: {{ .Values.env | quote }}
    managed-by: "namespace-factory"
  annotations:
    argocd.argoproj.io/sync-wave: "-2"
```
#### rbac-admin.yaml
This yaml give our tenant the edit  roleRef, which prevent the client from modifying our Quotas and RBAC making it a perfect fit for our tenants which will need permissions in their project and for us who want to retain our control over them.
Since the group is part of our LDAP we can easily use it to allow both our service account and our tenant to access the project.
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: {{ .Values.team }}-{{ .Values.project }}-admin-binding
  namespace: {{ .Values.team }}-{{ .Values.project }}-{{ .Values.env }}
  labels:
    team: {{ .Values.team | quote }}
    project: {{ .Values.project | quote }}
    env: {{ .Values.env | quote }}
    managed-by: "namespace-factory"
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: edit
subjects:
  - kind: Group
    name: {{ .Values.rbac.groups.admin | quote }}
    apiGroup: rbac.authorization.k8s.io
  - kind: ServiceAccount
    name: {{ .Values.project }}-pipeline-deployer
    namespace: {{ .Values.team }}-{{ .Values.project }}-{{ .Values.env }}
```
#### rbac-viewer.yaml
This role will be created only if the viewer variable is defined and will allow this group to view the project's resources.
```yaml
{{- if .Values.rbac.groups.viewer }}
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: {{ .Values.team }}-{{ .Values.project }}-view-binding
  namespace: {{ .Values.team }}-{{ .Values.project }}-{{ .Values.env }}
  labels:
    team: {{ .Values.team | quote }}
    project: {{ .Values.project | quote }}
    env: {{ .Values.env | quote }}
    managed-by: "namespace-factory"
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: view
subjects:
  - kind: Group
    name: {{ .Values.rbac.groups.viewer | quote }}
    apiGroup: rbac.authorization.k8s.io
{{- end }}
```


### Usage
Now that everything is set our directory folder should look like so:
```
.
├── README.md
├── charts
│   ├── project-template
│   │   ├── Chart.yaml
│   │   └── templates
│   │       └── app-project.yaml
│   └── tenant-namespace
│       ├── Chart.yaml
│       ├── templates
│       │   ├── limit-range.yaml
│       │   ├── namespace.yaml
│       │   ├── networkpolicy.yaml
│       │   ├── rbac-admin.yaml
│       │   ├── rbac-viewer.yaml
│       │   ├── resource-quota.yaml
│       │   └── serviceAccount.yaml
│       └── values.yaml
├── envs
│   ├── dev
│   │   └── alpha
│   │       └── myproj.yaml
│   └── prd
│       └── alpha
│           ├── myproj.yaml
│           ├── test.yaml
│           └── test2.yaml
├── root-app.yaml
├── team-app-projects.yaml
└── team-applications.yaml
```
and its time to talk about the day to day usage

to create a new project/team we will follow the following steps:
- choose our environment
- create the team's folder in case it does not exist
- create the project.yaml
- wait for the repo to sync automatically in 60 seconds and it will create the new project
- notify the team
