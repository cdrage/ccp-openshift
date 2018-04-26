To spin up things in an OpenShift cluster based on the contents in this
repository, please make sure you have a
[minishift](https://github.com/minishift/minishift/) based VM started up. We've
not tested this against anything other than minishift.

We used following command to start a minishift environment:

```bash
$ minishift start --disk-size 50GB --memory 8GB --iso-url centos --openshift-version 3.9.0
```

But the resources can be varied based on availability. However, make sure to
use `--iso-url centos` part in above command as we have setup things on CentOS
based minishift VM.

Once the VM is ready, spin up a Jenkins server that can be used by the Jenkins
Pipeline buildconfigs. Also, since we're going to be building images using
Jenkins pods, we need to add few capabilities to the Jenkins service account.
Do this on host system:

```bash
$ oc login -u system:admin
$ oc process -p MEMORY_LIMIT=1Gi openshift//jenkins-persistent| oc create -f -
$ oc adm policy add-scc-to-user privileged system:serviceaccount:myproject:jenkins
$ oc adm policy add-role-to-user system:image-builder system:serviceaccount:myproject:jenkins
```

This spins up a persistent Jenkins deployment which has 1 GB memory alloted to
it. The Jenkins service spun up by this template is recognized and used by the
Jenkins Pipelines.

Now, clone this repo on host system (not the VM). Login to the OpenShift
cluster as user `developer` and create a build from the buildconfig under
`seed-job` directory:

```bash
# on host system
$ git clone https://github.com/dharmit/ccp-openshift/
$ cd ccp-openshift
$ oc login -u developer
<use any password>
$ oc create -f seed-job/buildconfig.yaml
```

Now check in the OpenShift web console under Build -> Pipelines and see if a
Jenkins Pipeline has been created. Be patient because the image being used is
quite large (2.2 GB) at the moment.

To be able to build multiple container images at the same time, edit the
Jenkins deployment and add an environment variable `JENKINS_JAVA_OVERRIDES` to
it with the value
`-Dhudson.slaves.NodeProvisioner.initialDelay=0,-Dhudson.slaves.NodeProvisioner.MARGIN=50,-Dhudson.slaves.NodeProvisioner.MARGIN0=0.85`.

Since you changed the configuration, wait for the a new deployment to take
effect. Once it's done, exec into the Jenkins pod and check the output of `ps
-ef`. The three configuration options we added above should up in the `java`
command as space-separated and not comma-separated. Refer [this
diff](https://github.com/openshift/openshift-docs/pull/7259/files?short_path=05f80f3#diff-05f80f3ab954ce57c630417065819109)
to ensure that values are passed properly.