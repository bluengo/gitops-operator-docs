# Manual install (GitOps operator)


## 1. Gather a GitOps operator Nightly build tag:
(Jenkins in CPaaS by now)

## 2. Install GitOps operator as SDK:

## 3. Move to source directory:
```bash
cd ~/go/src/github.com/redhat-developer/gitops-operator
```

## 4. Deploy it with the make target:
```bash
make deploy  IMG=registry-proxy.engineering.redhat.com/rh-osbs/openshift-gitops-1-gitops-rhel8-operator:gitops-nightly-1-rhel-8-candidate-75384-20221208224222
```

