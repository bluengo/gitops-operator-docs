# Manual install (GitOps operator)


## 1. Gather a GitOps operator Nightly build tag:
(Jenkins in CPaaS by now)

## 2. Install GitOps operator as Go package:
```bash
# TODO: REVIEW
[bluengop@fedora ~]$ go install github.com/redhat-developer/gitops-operator@latest
go: downloading github.com/redhat-developer/gitops-operator v1.5.4
go: github.com/redhat-developer/gitops-operator@latest (in github.com/redhat-developer/gitops-operator@v1.5.4):
	The go.mod file for the module providing named packages contains one or
	more replace directives. It must not contain directives that would cause
	it to be interpreted differently than if it were the main module.

```

## 3. Move to source directory:
```bash
cd ~/go/src/github.com/redhat-developer/gitops-operator
```

## 4. Deploy it with the make target:
```bash
make deploy  IMG=registry-proxy.engineering.redhat.com/rh-osbs/openshift-gitops-1-gitops-rhel8-operator:gitops-nightly-1-rhel-8-candidate-75384-20221208224222
```

