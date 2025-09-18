# Deploying RHEL LightSpeed AI model on OpenShift

## Create Namespace
```console
oc new-project rhel-lightspeed
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
