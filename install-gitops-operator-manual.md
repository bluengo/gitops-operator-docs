# Manual installation of the GitOps operator
Follow this guide to manually install gitops core operator from a nightly build.

### Quick Start:

1. Get a GitOps operator Nightly build image ([steps here](#get-a-gitops-operator-nightly-build-tag)). Ensure your OCP cluster has [access to it]().
2. Clone GitOps operator repo:
    ```bash
    git clone git@github.com:redhat-developer/gitops-operator \
              ${GOPATH}/src/github.com/redhat-developer
    ```
3. Run make target to install the core operator. You just need to provide the image obtained in *Step 1* as `IMG` variable, either in the env or as a parameter, and then run the make target.
  i.e.:
    ```bash
    cd ${GOPATH}/src/github.com/redhat-developer
    make deploy IMG=registry-proxy.engineering.redhat.com/rh-osbs/openshift-gitops-1-gitops-rhel8-operator:gitops-nightly-1-rhel-8-candidate-75384-20221208224222
    ```
<br/>

---
### Steps:
* #### Get a GitOps operator Nightly build tag:
    
  You can get the operator bundle image from either the Brew tool or from CPaaS Jenkins build info:

  * **Option A**: 
    1. Check Brew to see the builds for package `openshift-gitops-operator-bundle-container`
    [brewweb.engineering.redhat.com/ ...](https://brewweb.engineering.redhat.com/brew/packageinfo?packageID=79092)
    <sup>(PackageID: **79092**)</sup>

    2. Look for the pull information.
    i.e.:

      ```yaml
      'pull': [
              'registry-proxy.engineering.redhat.com/rh-osbs/openshift-gitops-1-gitops-operator-bundle@sha256:20094c3733fadd3f07f747eaccfb087c26413bf08b869265d0190731f4cdea34',
              'registry-proxy.engineering.redhat.com/rh-osbs/openshift-gitops-1-gitops-operator-bundle:v99.9.0-11'
              ]
      ```
    <p><sup>(both container tags will work and behave the same)</sup></p>

  * **Option B**:
  <sub>(Use this whenever Brew system is unaccessible)</sub>

    1. Go to CPaaS Jenkins to get the build information from the last job:
     [jenkins-cpaas-openshift-gitops.apps.cpaas-poc.r6c9.p1.openshiftapps.com/job/gitops-nightly-1-rhel-8/ ...](https://jenkins-cpaas-openshift-gitops.apps.cpaas-poc.r6c9.p1.openshiftapps.com/job/gitops-nightly-1-rhel-8/job/build-pipeline)

    2. Copy or open the file `build-summary.json`.
    3. Look for `"componentName": "gitops-operator-bundle"`.
    4. Decode the content of the field `"build"` (base64 -d).
    5. Go to `"artifacts"` and get the info in the field `"nvr"`.
    6. The container tag is after the string *"openshift-gitops-operator-bundle-container-"*.
      i.e.:
        ```bash
        openshift-gitops-operator-bundle-container-v99.9.0-10
        # "v99.9.0-10" is what we're looking for
        ```
     7. Now append this tag to
      `registry-proxy.engineering.redhat.com/rh-osbs/openshift-gitops-1-gitops-operator-bundle:`
      to form the full container tag.
      i.e.:
      `registry-proxy.engineering.redhat.com/rh-osbs/openshift-gitops-1-gitops-operator-bundle:v99.9.0-10`
<br/>
        This snippet may help you:
        ```bash
        cat build-summary.json \
          | yq '.[] | select(.componentName == "gitops-operator-bundle") | .build' \
          | tr -d '"' \
          | base64 -d \
          | jq .artifacts[0].nvr \
          | egrep -o 'v[0-9]{,}\.[0-9]{,}\.[0-9]{,}\-[0-9]{,}'

        # yq -> looks for the "build" field in the component "gitops-operator-bundle"
        # tr -> removes quotes for base64
        # base64 -> decodes content of the build job
        # jq -> looks for "nvr" field in the build job information
        # egrep -> print only REGEX for format "v00.00.00-00"

        # example output
        # "v99.9.0-10"
        ```

* #### Configure access from OCP cluster to Image:
  They are diferent strategies to make the image available. You can mirror the image to a public registry and use a `ImageContentSourcePolicy` as [we do in OLM installation](#TODO: link to the olm doc).
  Here, instead, we will explain how to import certificate chain from redhat's registry to your OCP cluster.
  
  1. Import certificate chain to file:
  ```bash
  cd /tmp
  openssl s_client -showcerts -servername registry-proxy.engineering.redhat.com \
    -connect registry-proxy.engineering.redhat.com:443 </dev/null \
    | sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' > registry-proxy.engineering.redhat.com.ca.crt
  ```
  2. Create a `ConfigMap` containing a key with the name of the registry and the above file content:
  ```bash
  oc create configmap additional-registries-ca \
    -n openshift-config \
    --from-file=registry-proxy.engineering.redhat.com..443=registry-proxy.engineering.redhat.com.ca.crt \
    --from-file=registry-proxy.engineering.redhat.com=registry-proxy.engineering.redhat.com.ca.crt 
  ```
  3. Add the `ConfigMap` to the cluster image config by a patch merge command:
  ```bash
  oc patch image.config.openshift.io/cluster \
    --patch '{"spec":{"additionalTrustedCA":{"name":"additional-registries-ca"}}}' \
    --type=merge
  ```

---
##### Apendix A.

TODO: Detail below information
Trace of the make target execution:
* Runs `controller-gen`
* Moves to `config/manager` 
* Creates manifests with `kustomize edit set image controller="${IMG}"`
* Deploys operator with `kustomize build config/default | kubectl apply -f -`,
  creating the below resources:
  * namespace/gitops-operator-system
  * customresourcedefinition.apiextensions.k8s.io/applications.argoproj.io
  * customresourcedefinition.apiextensions.k8s.io/applicationsets.argoproj.io
  * customresourcedefinition.apiextensions.k8s.io/appprojects.argoproj.io
  * customresourcedefinition.apiextensions.k8s.io/argocds.argoproj.io
  * customresourcedefinition.apiextensions.k8s.io/gitopsservices.pipelines.openshift.io
  * serviceaccount/gitops-operator-controller-manager
  * role.rbac.authorization.k8s.io/gitops-operator-leader-election-role
  * clusterrole.rbac.authorization.k8s.io/gitops-operator-manager-role
  * clusterrole.rbac.authorization.k8s.io/gitops-operator-metrics-reader
  * clusterrole.rbac.authorization.k8s.io/gitops-operator-proxy-role
  * rolebinding.rbac.authorization.k8s.io/gitops-operator-leader-election-rolebinding
  * clusterrolebinding.rbac.authorization.k8s.io/gitops-operator-manager-rolebinding
  * clusterrolebinding.rbac.authorization.k8s.io/gitops-operator-proxy-rolebinding
  * configmap/gitops-operator-manager-config
  * service/gitops-operator-controller-manager-metrics-service
  * deployment.apps/gitops-operator-controller-manager
