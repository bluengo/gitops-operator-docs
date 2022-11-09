# Install GitOps Operator using OLM:


## 1. Prepare IIB image:

#### 1.1. Pull from registry-proxy
```bash
podman pull "registry-proxy.engineering.redhat.com/rh-osbs/iib:${IIB_ID}"
```

#### 1.2. Tag to public Quay
```bash
podman tag "registry-proxy.engineering.redhat.com/rh-osbs/iib:${IIB_ID}" "quay.io/${QUAY_USER}/iib:${IIB_ID}"
```

#### 1.3. Push to Quay
```bash
podman push "quay.io/${QUAY_USER}/iib:${IIB_ID}"
```


## 2. Give the cluster access to brew registry

#### 2.1. Export current pull-secret information
```bash
oc get secrets pull-secret -n openshift-config -o template='{{index .data ".dockerconfigjson"}}' | base64 -d > authfile
```

#### 2.2. Save login information for brew
```bash
podman login --authfile=authfile --username "|shared-qe-temp.src5.75b4d5" brew.registry.redhat.io
```

#### 2.3. Update the pull-secret information
```bash
oc set data secret/pull-secret -n openshift-config --from-file=.dockerconfigjson=authfile
```

This is a helper function example:

```bash
add_brew_registry_credentials ()
{
  local oldauth=$(mktemp)
  local newauth=$(mktemp)
  
  # Get current information
  oc get secrets pull-secret \
    -n openshift-config \
    -o template='{{index .data ".dockerconfigjson"}}' \
    | base64 -d > ${oldauth}
  
  # Get Brew registry credentials
  local brew_secret=$(jq '.auths."brew.registry.redhat.io".auth' \
                      ${HOME}/.docker/config.json \
                      | tr -d '"')

  # Append the key:value to the JSON file
  jq --arg secret ${brew_secret} \
    '.auths |= . + {"brew.registry.redhat.io":{"auth":$secret}}' \
    ${oldauth} > ${newauth}

  # Update the pull-secret information in OCP
  oc set data secret pull-secret \
    -n openshift-config \
    --from-file=.dockerconfigjson=${newauth}

  # Cleanup
  rm -f ${oldauth} ${newauth}
}
```


## 3. Apply manifests

#### 3.1. An ImageContentSourcePolicy to point to brew registry for all images specified in IIB bundle
```bash
cat << EOF | oc apply -f -
apiVersion: operator.openshift.io/v1alpha1 
kind: ImageContentSourcePolicy 
metadata: 
  name: brew-registry 
spec: 
  repositoryDigestMirrors: 
  - mirrors: 
    - brew.registry.redhat.io 
    source: registry.redhat.io 
  - mirrors: 
    - brew.registry.redhat.io 
    source: registry.stage.redhat.io 
  - mirrors: 
    - brew.registry.redhat.io 
    source: registry-proxy.engineering.redhat.com
EOF
```

#### 3.2. A new CatalogSource to make the new source available in the "marketplace"
```bash
cat << EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: iib-${QUAY_USER}
  namespace: openshift-marketplace
spec:
  sourceType: grpc
  image: quay.io/${QUAY_USER}/iib:${IIB_ID}
  displayName: iib-${QUAY_USER}
  publisher: GitOps Team
EOF
```

#### 3.3. A new subscription to install the operator
```bash
cat << EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-gitops-operator
  namespace: openshift-operators
spec:
  channel: gitops-${GITOPS_VERSION}
  installPlanApproval: Automatic
  name: openshift-gitops-operator
  source: iib-${QUAY_USER}
  sourceNamespace: openshift-marketplace
EOF
```

