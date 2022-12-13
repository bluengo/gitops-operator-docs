# Manual install (GitOps operator)
Follow this guide to manually install GitOps operator from source code.

## * Quick Start:

### 1. Gather a GitOps operator Nightly build tag:
* Go to CPaaS Jenkins to get the build information from the last job:
https://jenkins-cpaas-openshift-gitops.apps.cpaas-poc.r6c9.p1.openshiftapps.com/job/gitops-nightly-1-rhel-8/job/build-pipeline
* Or either, check Brew to see the builds for package "`openshift-gitops-operator-bundle-container`" (PackageID: 79092)
https://brewweb.engineering.redhat.com/brew/packageinfo?packageID=79092

You need to look for the pull information.
i.e.:
```yaml
'pull': ['registry-proxy.engineering.redhat.com/rh-osbs/openshift-gitops-1-gitops-operator-bundle@sha256:20094c3733fadd3f07f747eaccfb087c26413bf08b869265d0190731f4cdea34',
                              'registry-proxy.engineering.redhat.com/rh-osbs/openshift-gitops-1-gitops-operator-bundle:v99.9.0-11']
```
(both container tags will work and behave the same)

### 2. Clone GitOps operator repository:
```bash
git clone github.com/redhat-developer/gitops-operator \
          "${GOPATH}/src/github.com/redhat-developer/gitops-operator"
cd gitops-operator
```

### 3. Deploy it with the make target:
You just need to provide the image obtained in Step 1 as `IMG` variable, either in the env or as a parameter, and then run the make target.
i.e.:
```bash
make deploy IMG=registry-proxy.engineering.redhat.com/rh-osbs/openshift-gitops-1-gitops-rhel8-operator:gitops-nightly-1-rhel-8-candidate-75384-20221208224222
```

