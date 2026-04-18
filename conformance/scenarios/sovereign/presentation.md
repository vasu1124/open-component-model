---
options:
  implicit_slide_ends: true
title: "Sovereign Scenario Walkthrough"
sub_title: "Deliver your product ubiquitously"

theme:
  override:
    code:
      alignment: left
      background: true
---

# Setup a Sovereign "Airgapped" Cluster

```bash +exec +pty:120:30
task check 
task clean 
task prepare
task cluster:setup
```
<!-- end_slide -->

# Cache the full Sovereign Demo

```bash +exec +pty:120:30
task build:product
task transfer:airgap
task cluster:import
task cluster:bootstrap
task verify:deployment
sleep 30
# upgrade
task upgrade
```
<!-- end_slide -->

# Reset for Interactive Demoflow

```bash +exec +pty:120:30
# reset for demoflow
kubectl delete -f deploy/sample-product-1.1.0.yaml --timeout 20s --force
sleep 10
kubectl delete -f deploy/bootstrap.yaml --timeout 20s
sleep 10
kubectl delete -f components/product/deploy/rgd.yaml --timeout 20s
```

Often, you must manually delete the resources created by the RGD.

<!-- end_slide -->

# OCM Component Constructor for Product - orchestrates Notes and Postgres

## component-constructor.yaml
```yaml +line_numbers {1-2|1,3|1,4-5|1-5|8-11|8,12-14|16-24}
components:
  - name: acme.org/sovereign/product
    version: "1.0.0"
    provider:
      name: ocm.software/sovereign

    # References to child components
    componentReferences:
      - name: notes
        componentName: acme.org/sovereign/notes
        version: "1.0.0"
      - name: postgres
        componentName: acme.org/sovereign/postgres
        version: "1.0.0"

    resources:
      # Product-level RGD as Operator for orchestrating deployment order
      - name: product-rgd
        type: blob
        relation: local
        input:
          type: file
          path: ./deploy/rgd.yaml
          mediaType: application/vnd.cncf.kro.resourcegraphdefinition.v1+yaml
```
<!-- end_slide -->

# OCM Component Constructor for PostgreSQL

## component-constructor.yaml
```yaml +line_numbers {1-22|12-14}
components:
  - name: acme.org/sovereign/postgres
    version: "1.0.0"
    provider:
      name: ocm.software/sovereign
    
    resources:
      # Official PostgreSQL image
      - name: image
        type: ociImage
        version: "18"
        access:
          type: ociArtifact
          imageReference: docker.io/library/postgres:18

      # Helm chart for StatefulSet deployment
      - name: helm-chart
        type: helmChart
        relation: local
        input:
          type: helm
          path: ./deploy/chart
```
<!-- end_slide -->

# OCM Component Constructor for Notes

## component-constructor.yaml
```yaml +line_numbers {1-31|14-16}
components:
  - name: acme.org/sovereign/notes
    version: "1.0.0"
    provider:
      name: ocm.software/sovereign
      
    resources:
      # The application container image (built from source)
      - name: image
        type: ociImage
        relation: local
        version: "1.0.0"
        input:
          type: file
          mediaType: application/vnd.ocm.software.oci.layout.v1+tar+gzip
          path: "./deploy/notes.tar.gz"

      # Helm chart for Kubernetes deployment
      - name: helm-chart
        type: helmChart
        relation: local
        input:
          type: helm
          path: ./deploy/chart
          
      # OpenAPI specification for the notes API
      - name: openapi-spec
      ...
      # Open Resource Discovery metadata
      - name: ord-document
      ...
```
<!-- end_slide -->

# Build the Component Notes

```bash
task build:notes
ocm add componentversion \
    --repository ctf::./tmp/transport-archive \
    ... \
    --constructor components/notes/component-constructor.yaml
```

```bash +exec +pty:120:30
ocm get resources ctf::./tmp/transport-archive//acme.org/sovereign/notes:1.0.0 -o tree
echo ========
ocm get componentversions ctf::./tmp/transport-archive//acme.org/sovereign/notes:1.0.0 --output tree -r
```
<!-- end_slide -->

# Build the Component Postgres

```bash
task build:postgres
ocm add componentversion \
    --repository ctf::./tmp/transport-archive \
    ... \
    --constructor components/postgres/component-constructor.yaml
```

```bash +exec +pty:120:30
ocm get resources ctf::./tmp/transport-archive//acme.org/sovereign/postgres:1.0.0 -o tree
echo ========
ocm get componentversions ctf::./tmp/transport-archive//acme.org/sovereign/postgres:1.0.0 --output tree -r
```
<!-- end_slide -->

# Build the Product and Sign

```bash
task build:product
# ...
ocm add componentversion \
    --repository ctf::./tmp/transport-archive \
    ... \
    --constructor components/product/component-constructor.yaml
 
# ... create acme.org keys etc.
ocm sign componentversion components/product
```

```bash +exec +pty:120:30
ocm get resources ctf::./tmp/transport-archive//acme.org/sovereign/product:1.0.0 -o tree
echo ========
ocm get componentversions ctf::./tmp/transport-archive//acme.org/sovereign/product:1.0.0 -o tree -r
```
<!-- end_slide -->

# The generated OCM Product component 

```bash +exec +pty:120:30 +acquire_terminal
ocm get componentversions ctf::./tmp/transport-archive//acme.org/sovereign/product:1.0.0 -o yaml | 
  bat --number -l yaml --paging=always
```
<!-- end_slide -->

# I like Kubernetes YAML

```bash +exec +pty:120:30 +acquire_terminal
ocm get componentversions ctf::./tmp/transport-archive//acme.org/sovereign/product:1.0.0 -o yaml -S v3alpha1 |
  bat --number -l yaml --paging=always
```
<!-- end_slide -->

# Transfer the Product to Airgap Archive

```bash
# task transfer:airgap

ocm transfer componentversions \
  --copy-resources \
  --recursive \
  ctf::./tmp/transport-archive//acme.org/sovereign/product:1.0.0 \
  ctf::./tmp/airgap-archive

transferring version "acme.org/sovereign/product:1.0.0"...
  transferring version "acme.org/sovereign/notes:1.0.0"...
  ...resource 0 image[ociImage]...
  ...resource 1 helm-chart[helmChart]...
  ...resource 2 openapi-spec[blob]...
  ...resource 3 ord-document[blob]...
  ...adding component version...
  transferring version "acme.org/sovereign/postgres:1.0.0"...
  ...resource 0 image[ociImage](library/postgres:18)...
  ...resource 1 helm-chart[helmChart]...
  ...adding component version...
...resource 0 product-rgd[blob]...
...adding component version...
3 versions transferred
```
```bash +exec +pty:120:30
du -sh tmp/transport-archive
echo ========
du -sh tmp/airgap-archive
```

<!-- end_slide -->

# Transfer the Product to Registry

```bash
# task cluster:import
ocm transfer componentversions \
  --recursive \
  --copy-resources \
  ctf::./tmp/airgap-archive//acme.org/sovereign/product:1.0.0 \
  http://127.0.0.1:5001

transferring version "acme.org/sovereign/product:1.0.0"...
  transferring version "acme.org/sovereign/notes:1.0.0"...
  ...resource 0 image[ociImage]...
  ...resource 1 helm-chart[helmChart]...
  ...resource 2 openapi-spec[blob]...
  ...resource 3 ord-document[blob]...
  ...adding component version...
  transferring version "acme.org/sovereign/postgres:1.0.0"...
  ...resource 0 image[ociImage](library/postgres:18)...
  ...resource 1 helm-chart[helmChart]...
  ...adding component version...
...resource 0 product-rgd[blob]...
...adding component version...
3 versions transferred
```
```bash +exec +pty:120:30
ocm get componentversions http://127.0.0.1:5001//acme.org/sovereign/product:1.0.0 -o tree -r
echo =======
ocm get resources http://127.0.0.1:5001//acme.org/sovereign/product:1.0.0 -o tree -r
```
<!-- end_slide -->

# Bootstrap with OCM Controllers (Repository)
## OCM Repository pointing to local air-gapped registry

```yaml +line_numbers
apiVersion: delivery.ocm.software/v1alpha1
kind: Repository
metadata:
  name: sovereign-repo
  namespace: sovereign-product
spec:
  repositorySpec:
    baseUrl: http://registry:5000
    type: OCIRegistry
  interval: 10m
```
<!-- end_slide -->

# Bootstrap with OCM Controllers (Component)
## OCM Component Version from air-gapped registry and signature verification

```yaml +line_numbers
apiVersion: delivery.ocm.software/v1alpha1
kind: Component
metadata:
  name: sovereign-product-component
  namespace: sovereign-product
spec:
  component: acme.org/sovereign/product
  repositoryRef:
    name: sovereign-repo
  semver: "1.0.0"
  interval: 1m
  verify:
    - signature: default
      secretRef:
        name: acme-signing-key
```
```bash
# PKI
kubectl -n sovereign-product create secret generic acme-signing-key \
        --from-file=default=keys/acme-public.pem
```

The supply chain can also be automatically triggered as soon as a new version is available in the registry.
For minor updates/patches or yolo major upgrades. 
For example:
* `semver: ">=1.0.x"` //automatically install patch versions >=1.0.x
* `semverFilter: ".*-rc.*"` //skip pre-release candidates

<!-- end_slide -->

# Bootstrap with OCM Controllers (Resource)
## Linking to a particular Resource in a Component Version

```yaml +line_numbers
apiVersion: delivery.ocm.software/v1alpha1
kind: Resource
metadata:
  name: sovereign-product-resource-rgd
  namespace: sovereign-product
spec:
  componentRef:
    name: sovereign-product-component
  resource:
    byReference:
      resource:
        name: product-rgd
```

```
COMPONENT                         NAME         VERSION IDENTITY TYPE      RELATION
└─ acme.org/sovereign/product                  1.0.0
   └─                             product-rgd  1.0.0            blob      local      <==== this!
```
<!-- end_slide -->

# Bootstrap with OCM Controllers (Deployer)
## Automatic Deployment of that Resource 
The Deployer is extensible and works with simple K8s manifests.
We advise/prefer to orchestrate with kro and RGDs, or a proper Operator. 
This abstracts Helm and any other package and deployment tooling behind a declarative API surface.

```yaml +line_numbers
apiVersion: delivery.ocm.software/v1alpha1
kind: Deployer
metadata:
  name: sovereign-product-deployer
  namespace: sovereign-product
spec:
  resourceRef:
    name: sovereign-product-resource-rgd
    namespace: sovereign-product
```
```bash +exec +pty:120:30
kubectl apply -f deploy/bootstrap.yaml
```

<!-- end_slide -->

# Install the Sovereign Product

```yaml
apiVersion: kro.run/v1alpha1
kind: SovereignProduct
metadata:
  name: sovereign-product
  namespace: sovereign-product
spec:
  version: "1.0.0"
  repository:
    baseUrl: http://registry:5000
    type: OCIRegistry
  notes:
    replicas: 2
    resources:
      requests:
        cpu: "100m"
        memory: "64Mi"
      limits:
        cpu: "200m"
        memory: "128Mi"
  postgres:
    database:
      name: "sovereign_notes"
      username: "sovereign_user"
      # NOTE: Demo-only default. Production should use ESO or similar.
      password: "changeme"
    persistence:
      size: "8Gi"
    resources:
      requests:
        cpu: "250m"
        memory: "256Mi"
      limits:
        cpu: "500m"
        memory: "512Mi"
```
```bash +exec +pty:120:30
kubectl apply -f deploy/sample-product-1.0.0.yaml
```

<!-- end_slide -->

# Product 1.0.0 is running in the Cluster

```bash +exec +pty:120:30
kubectl port-forward -n sovereign-product services/sovereign-notes 8080:80
```

<!-- end_slide -->

# Upgrade the Product 1.0.0 to 1.1.0

```bash +exec +pty:120:30
# task upgrade
ocm get componentversions http://127.0.0.1:5001//acme.org/sovereign/product -r -o tree
```

<!-- end_slide -->

# Upgrade the Product 1.0.0 to 1.1.0 in the Cluster

```bash +exec +pty:120:30
kubectl apply -f deploy/sample-product-1.1.0.yaml
```

<!-- end_slide -->

# End of Scenario Walkthrough

```bash +exec +pty:120:30
kubectl port-forward -n sovereign-product services/sovereign-notes 8080:80
```

The Sovereign Product represents successful air-gapped component modeling, cryptographic verification, transport abstraction, and Operator-like rollouts for day 1 and 2 via custom CRDs using OCM, kro and Flux.
