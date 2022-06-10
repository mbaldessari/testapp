# Refresh / sync with helm oddity

## (A) Install OCP 4.10.15

Go to OperatorHub and install Red Hat Openshift Gitops (currently at v1.5.2)
Leave default (namespace to openshift-operators).
## (B) Test argocd from git upstream v2.4.0-rc5+b84dd8b

Git clone https://github.com/mbaldessari/argocd-operator/tree/2.4.0-rc5 which will install
one of the latest argocd containers.

Set KUBECONFIG and Hop in the `argocd-operator` folder and run:
```
IMAGE_TAG_BASE=quay.io/rhn_support_mbaldess/argocd-operator VERSION=0.6.6 \
IMG=quay.io/rhn_support_mbaldess/argocd-operator:"${VERSION}" \
CHANNELS=fast make generate bundle docker-build docker-push bundle deploy
```

This will install the operator on our cluster. Then run the following which will create
the `openshift-gitops` namespace and put an argo instance in there:
```
oc apply -f templates/argocd-basic.yaml
```

# Testing protocol

The following testing protocol reproduces the issue on both (A) OCP + gitops 1.5.2 *and* (B) OCP + ArgoCDv2.4.0-rc5+b84dd8b

Login via the argocd cli as it makes things easier:
```
argocd login $(oc get routes -n openshift-gitops openshift-gitops-server -o=jsonpath='{ .spec.host }') --sso
```

```
# We create project and application
$ oc apply -f apps/test/templates/project.yaml
$ oc apply -f apps/test/templates/application.yaml

# This is our correct starting configmap exporting all helm values:
$ oc get -n openshift-gitops configmap/configmap-test -o yaml

kind: ConfigMap
  name: configmap-test
  namespace: openshift-gitops
apiVersion: v1
data:
  values.yaml: |
    clusterGroup:
      foo:
        bandini1: 222
        bandini2: 456
        baz: 234
        jobs:
        - name: foo
          playbook: foo.yml
        - name: bar
          playbook: bar.yml
      test: 111
    global:
      hubClusterDomain: apps.bandini-dc.blueprints.rhecoeng.com
      localClusterDomain: apps.bandini-dc.blueprints.rhecoeng.com
      namespace: openshift-gitops
      options:
        installPlanApproval: Automatic
        syncPolicy: Automatic
        useCSV: false
      pattern: testapp
      repoURL: https://github.com/mbaldessari/testapp.git
      targetRevision: HEAD
      valuesDirectoryURL: https://github.com/mbaldessari/testapp/raw/main
    main:
      clusterGroupName: hub
```

So now we update `values-override.yaml` and add `bandini3: 789` undercloud clusterGroup.foo and push the change to the git origin.
It seems that whenever we do the *first* change it all works without issues and we see it all reflected in the configmap:
```
$ oc get -n openshift-gitops configmap/configmap-test -o yaml |grep bandini3 |grep -v '{'
        bandini3: 789
```

Now the issue seems to be happening on the *second* update. Namely now I update the values-override.yaml
file and add `bandini4: 000`, so now we have:
```
clusterGroup:
  test: 111
  foo:
    bandini1: 222
    bandini2: 456
    bandini3: 789
    bandini4: 000
    baz: 234
    jobs:
      - name: foo
        playbook: foo.yml
      - name: bar
        playbook: bar.yml
```

After clicking on refresh and sync there is no trace of bandini4 in the configmap:
```
$ oc get -n openshift-gitops configmap/configmap-test -o yaml |grep bandini4 |grep -v '{'
$
```

There is no amount of refresh + sync that will bring in bandini4 in the configmap. Note that
sometimes (? not always it seems) bandini4 will show up in the parameters of the application
manifest (but never in the configmap):

  clusterGroup.foo.bandini4: 0

## Workaround

Now the interesting part seems to be that a `argocd app get test --hard-refresh` will fix things
and the configmap will now contain the expected bandini4 key. At this point it seems no change
will ever be reflected on the configmap in k8s, unless I hard-refresh via the CLI.
Looking at the definition of refresh vs hard-refresh:
* Sync: Reconciles the current cluster state with the target state in git.
* Refresh: Fetches the latest manifests from git and compares diff.
* Hard Refresh: Argo CD caches the manifests and a hard refresh will invalidate this cache.

So it seems to me that somehow this is a cache invalidation problem of sorts.

## Similar reports

* https://github.com/argoproj/argo-cd/issues/9214 - argocd application do not update when values file changed in helm repo
