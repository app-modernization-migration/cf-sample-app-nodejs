# Node.js Sample App

From your Cloud Foundry environment extract the configuration needed for this application:

```cf env cf-sample-app-nodejs```

Save that information (a JSON file) into the [`cf-vcap-application.json`](./cloudfoundry/cd-vcap-application.json) file
to be used later.

## Deploying in OpenShift

Create a new project:

```shell
oc new-project cf2ocp-sample-app-nodejs
```

Create the ConfigMap with the values from the original configuration:

```shell
oc create configmap cf-sample-app-nodejs-cm \
    --from-file=VCAP_APPLICATION=cloudfoundry/cf-vcap-application.json \
    --from-literal=CF_INSTANCE_INDEX=OpenShift
```

This application could be deployed by:

* Using [OpenShift Source-to-Image](https://docs.openshift.com/container-platform/4.9/openshift_images/using_images/using-s21-images.html)
* Using [OpenShift Pipelines](https://docs.openshift.com/container-platform/4.9/cicd/pipelines/understanding-openshift-pipelines.html)

### Using OpenShift Source-to-Image

Deploy the application the first time using OpenShift Developer capabilities:

```shell
oc new-app . -e PORT=8080
```

Add the original configuration into the new Deployment:

```shell
oc patch deployment/cf-sample-app-nodejs \
    -p '{"spec":{"template":{"spec":{"containers":[{"name":"cf-sample-app-nodejs","env":[{"name":"VCAP_APPLICATION","valueFrom":{"configMapKeyRef":{"name":"cf-sample-app-nodejs-cm","key":"VCAP_APPLICATION"}}},{"name":"CF_INSTANCE_INDEX","valueFrom":{"configMapKeyRef":{"name":"cf-sample-app-nodejs-cm","key":"CF_INSTANCE_INDEX"}}}]}]}}}}'
```

Expose the service to access from outside OpenShift:

```shell
oc expose svc/cf-sample-app-nodejs
```

The new route is:

```
curl http://$(oc get route cf-sample-app-nodejs -o jsonpath='{.spec.host}')/
```

### Using OpenShift Pipelines

Tekton Pipelines will allow us to declare a CICD pipeline of our application inside of OpenShift.

Deploy OpenShift pipelines operator (`cluster-admin` role is required):

```shell
oc apply -f tekton/pipelines-operator.yaml
```

Create a `pvc` to use in the pipeline:

```shell
oc apply -f tekton/pvc.yaml
```

Create a Tekton CICD pipeline to deploy the application:

```shell
oc apply -f tekton/pipeline.yaml
```

Check pipelines:

```shell
❯ tkn pipeline list
NAME                   AGE             LAST RUN   STARTED   DURATION   STATUS
cf-sample-app-nodejs   8 seconds ago   ---        ---       ---        ---
```

Start pipeline with the default configuration

```shell
tkn pipeline start cf-sample-app-nodejs --use-param-defaults -w name=workspace,claimName=tkn-pvc-cf-sample-app-nodejs
```

The new route is:

```
curl http://$(oc get route cf-sample-app-nodejs -o jsonpath='{.spec.host}')/
```
