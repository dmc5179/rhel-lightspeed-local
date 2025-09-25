# Deploying RHEL LightSpeed AI model on OpenShift

## Create Namespace
```console
oc new-project rhel-lightspeed
```

## Create SCC
```console
oc create -f 01-scc-postgres.yaml
oc create -f 02-serviceaccounts.yaml
oc create -f 03-rolebindings.yaml
```

## Allow anyuid
```console
oc adm policy add-scc-to-user -n rhel-lightspeed anyuid -z default
```

## Deploy on OpenShift
```
oc create -f rhel-lightspeed-deployment.yaml
```

## Create Route for Lightspeed model
```console
oc create -f lightspeed-route.yaml
```


# TODO

- Need an init container that copies the model into the "cuda" image.
  Copy it from the installer container or from where ever that would exist?
