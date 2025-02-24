# Kibosh

An [open service broker](https://github.com/openservicebrokerapi/servicebroker)
bridging the gap between Kubernetes deployments and CF marketplace.

When deployed with a Helm chart and added to the marketplace,
* `cf create-service` calls to Kibosh will create the collection of Kubernetes resources described by the chart.
* `cf bind-service` calls to Kibosh will expose back any services and secrets created by the chart

### Overriding/Setting values as defined in values.yaml via 'cf create-service' or 'cf update-service'.  

The format of the json string is a nested format.  
Also refer to the cf cli for an example of a valid JSON object.

**Example:** Setting the mysqlUser on `cf create-service` for the MySQL chart

values.yaml

```## Create a database user
##
# mysqlUser:
# mysqlPassword:
```

`cf create-service mysql medium mysql-kibosh-service -c '{"mysqlUser":"admin"}'`

**Example:** Setting the resources.requests.memory on `cf update-service` for the MySQL chart

values.yaml

```## Configure resource requests and limits
## ref: http://kubernetes.io/docs/user-guide/compute-resources/
##
resources:
  requests:
    memory: 256Mi
    cpu: 100m
```

`cf update-service mysql-kibosh-service  -c '{"resources": {"requests": {"memory": "256Mi"}}}'`

For some in depth discussion, see this blog post:
[Use Kubernetes Helm Packages to Build Pivotal Cloud Foundry tiles](https://content.pivotal.io/blog/use-kubernetes-helm-packages-to-build-pivotal-cloud-foundry-tiles-kibosh-a-new-service-broker-makes-it-simple)

![](docs/kibosh_logo_100.png)

## Configuration
### Changes required in Chart
#### Plans (`cf marketplace`)  
Kibosh requires that helm chart has additional file that describes plan in `plans.yaml` at root level

```yaml
- name: "small"
description: "default (small) plan for mysql"
file: "small.yaml"
- name: "medium"
description: "medium sized plan for mysql"
file: "medium.yaml"
```

* `file` is a filename that exists in the `plans` subdirectory of the chart.
* File names should consist of only lowercase letters, digits, `.`, or `-`.
* The standard `values.yaml` file in the helm chart sets the defaults.
* Each plan's yaml file is a set of values overriding the defaults present in `values.yaml`.  

Copy any key/value pairs to override from `values.yaml` into a new plan file and change their value.  
See kibosh-sample's [sample-charts](https://github.com/cf-platform-eng/kibosh-sample/tree/master/sample-charts) for a few examples.

In order to successfully pull private images, we're imposing some requirements
on the `values.yaml` file structure

* **Single-image** charts should use this structure:
    ```yaml
    ---
    image: "my-image"
    imageTag: "5.7.14"
    ```
* **Multi-image** charts shoud use this structure:
    ```yaml
    ---
    images:
      thing1:
        image: "my-first-image"
        imageTag: "5.7.14"
      thing2:
        image: "my-second-image"
        imageTag: "1.2.3"
    ```

### Plan-Specific Clusters
_This feature is experimental and the syntax will likely change in the future_

By default, Kibosh will create all deployments in the same cluster. It's also possible for each plan to 
target a different cluster. In `plans.yaml`, the plan specifies a credentials file:
```yaml
---
- name: "small"
  description: "default (small) plan for mysql"
  file: "small.yaml"
  credentials: "small-creds.yaml"
```
    
The contents of this file mirror what would appear in the `.kube/config` file. For example, `small-creds.yaml`
would contain:

```yaml
---
apiVersion: v1
clusters:
  - cluster:
      certificate-authority-data: bXktY2VydA==
      server: https://pks.example.com
    name: my-cluster
contexts:
  - context:
      cluster: my-cluster
      user: my-user
    name: my-cluster
current-context: my-cluster
kind: Config
preferences: {}
users:
  - name: my-user
    user:
      token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

### Bind Templates

Developers and libraries often have specific assumptions around how the bind
environment variable should be structured. For example,
[Spring Cloud Connectors](https://cloud.spring.io/spring-cloud-connectors/) will
automatically support a service if the 
[right structure is present](https://cloud.spring.io/spring-cloud-connectors/spring-cloud-cloud-foundry-connector.html#_mysql).

Chart authors can transform what the Kibosh broker returns by writing a
[Jsonnet](https://jsonnet.org) template and putting it in the `bind.yaml` file
in the root of the Chart. For example, the `bind.yaml` following will transform the bind
response into a readily consumable structure:

```yaml
template: |
  {
    hostname: $.services[0].status.loadBalancer.ingress[0].ip,
    name: $.services[0].name,
    jdbcUrl: "jdbc:mysql://" + self.hostname + "/my_db?user=" + self.username + "&password=" + self.password + "&useSSL=false",
    uri: "mysql://" + self.username + ":" + self.password + "@" + self.hostname + ":" + self.port + "/my_db?reconnect=true",
    password: $.secrets[0].data['mysql-root-password'],
    port: 3306,
    username: "root"
  }
```

The template is executed in an environment has top-level `services` and `secrets`,
which are json marshalled versions of the services and secrets in the namespace
generated for the service. 

To test your bind template, use the template-tester binary from the [github release.](https://github.com/cf-platform-eng/kibosh/releases/latest)
It takes the namespace in which you have already deployed your helm chart and the file that has the Jsonnet template descrited above.

```bash
template-tester mynamespaceid bind.yaml
```

### CredHub Integration
*Note: In order to follow the steps for [Credhub](https://docs.cloudfoundry.org/credhub/) integration, 
you should have some familiarity with [UAA](https://docs.run.pivotal.io/concepts/architecture/uaa.html) 
and [UAAC](https://github.com/cloudfoundry/cf-uaac)*

Kibosh can be configured to store binding credentials in [CredHub](https://docs.cloudfoundry.org/credhub/). 
To do so, include the following environment variables in the Kibosh configuration.

```
CH_CRED_HUB_URL: https://[credhub-url]
CH_UAA_URL: https://[credhub-uaa-url]
CH_UAA_CLIENT_NAME: [my-uaa-client]
CH_UAA_CLIENT_SECRET: [my-uaa-secret]
CH_SKIP_SSL_VALIDATION: true
```

#### Setting up the Client
Firstly, the Kibosh UAA client needs to created and given the correct scope in order to store credentials in Credhub. 
Use the [uaac](https://github.com/cloudfoundry/cf-uaac) cli to do so. You must know the UAA URL and UAA admin client secret.

If using the CF runtime credhub, the UAA URL is `uaa.[system-domain]`.
```bash
uaac target https://[credhub-uaa-url] --skip-ssl-validation
uaac token client get admin

uaac client add my-uaa-client \
    --access_token_validity 1209600 \
    --authorized_grant_types client_credentials,refresh_token \
    -s my-uaa-secret \
    --scope openid,oauth.approvals,credhub.read,credhub.write \
    --authorities oauth.login,credhub.read,credhub.write
```

Secondly, you must get the token for the Credhub Admin client. You must know the Credhub admin client secret to do so. 
If using PCF, you can find the Credhub admin client secret from "Credhub Admin Client Client Credentials" in Ops Manager.

```bash
uaac token client get credhub_admin_client
# get the token by viewing the context
uaac context
```

Finally, give our newly created client access to modify creds in CredHub. This can be done via curl.
```
curl -k "https://[credhub-url]/api/v2/permissions" \
  -X POST \
  -d '{
     "path": "/c/kibosh/*",
     "actor": "uaa-client:my-uaa-client",
     "operations": ["read", "write", "delete", "read_acl", "write_acl"]
  }' \
  -H "authorization: bearer [CREDHUB-ADMIN-CLIENT-TOKEN]" \
  -H 'content-type: application/json'
```

#### Testing with CF runtime Credhub

The CF runtime CredHub does not expose an external url, so testing with CredHub can be done by
* Running a proxy app on the platform to expose runtime Credhub url externally
* Pushing Kibosh broker as an app OR Running Kibosh in a BOSH release

To run a proxy, `cf push` the app located in [docs/credhub_proxy](docs/credhub_proxy). The proxy
will then make credhub available at `https://credhub-proxy.<cf apps domain>`. 

Then add the following set of environment variables to configure the Kibosh process:
```bash
CH_CRED_HUB_URL: https://credhub-proxy.[apps-domain]
CH_UAA_URL: https://uaa.[system-domain]
CH_UAA_CLIENT_NAME: my-uaa-client
CH_UAA_CLIENT_SECRET: my-uaa-secret
CH_SKIP_SSL_VALIDATION: true
```

To push the Kibosh broker as an app, use the [sample manifest](docs/sample-manifest.yaml) and run `cf push` from the Kibosh project root. 
Then run `cf create-service-broker SERVICE_BROKER USERNAME PASSWORD URL` to register the broker with CF. 

Once you provision and bind a service from Kibosh, running `cf env` agains the application should return a placeholder value to Credhub instead of the credentials in plain text. 

### Other Requirements

* When defining a `Service`, to expose this back to any applications that are bound,
  `type: LoadBalancer` is a current requirement.
   `NodePort` is also an option and Kibosh will add externalIPs and nodePort to bind json, but `NodePort` does carry significant risks and probably should **not** be used in production: is not robust to cluster scaling events, upgrades or other IP changes.
* Resizing disks has limitiations. To support upgrade:
    - You can't resize a persistent volume claim (currently behind an [alpha feature gate](https://kubernetes.io/docs/reference/feature-gates/))
* Selectors are [immutable](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#selector)
    - This means that *chart name cannot change* (the name is generally used in selectors)

### Private registries
When the environment settings for a private registry are present (`REG_SERVER`, `REG_USER`, `REG_PASS`), 
then Kibosh will transform images to pull them from the private registry. It assumes
the image is already present (see the Kibosh deployment). It will patch
the default service account in the instance namespaces to add in the registry credentials.

Be sure that `REG_SERVER` contains any required path information. For example, in gcp `gcr.io/my-project-name`

## Contributing to Kibosh

We welcome comments, questions, and contributions from community members. Please consider
the following ways to contribute:

* File Github issues for questions, bugs and new features and comment and vote on the ones that you are interested in.
* If you want to contribute code, please make your code changes on a fork of this repository and submit a
pull request to the master branch of Kibosh. We strongly suggest that you first file an issue to
let us know of your intent, or comment on the issue you are planning to address.

## Deploying
To manually deploy the BOSH release, get the latest BOSH release (`kibosh-release-X.X.XX.tgz`)
from the  [Github releases](https://github.com/cf-platform-eng/kibosh/releases) and upload
to your director.

Build a manifest by starting from the example bosh-lite manifest 
[lite-manifest.yml](bosh/bosh-release/manifests/lite-bazaar-manifest.yml)
and customize the cloud specific settings (`az`, `vm_type`, etc). This manifest
uses a set of [input variables](https://bosh.io/docs/cli-int/).
See
[values-sample.yml](bosh/bosh-release/manifests/values-sample.yml)
for example values.

## Dev
### Setup

Clone the repo (recommended to clone outside of `$GOPATH` because using go modules).

### Run
Run `make bootstrap` from a clean checkout to setup initial dependencies. This will set up the dependencies
in the `go.sum` file as well as get external binary tools (counterfeiter, ginkgo, etc).

#### For a remote K8s Cluster
Copy `local_dev.sh.template` to `local_dev.sh` (which is in `.gitignore`) and 
configure the values (`cluster.certificate-authority-data`, `cluster.server`, and `user.token`)
for a working cluster. Then run:

```bash
./local_dev.sh
```

#### For minikube
Make sure minikube is running:

```bash
minikube start --vm-driver=hyperkit
```

Use `local_dev_minikube` to set up all the secrets and start kibosh:

```bash
local_dev_minikube.sh
```

### Securing tiller
In production, tiller **should be secured**. It's probably good practice to use secure tiller
in your local environment as well (at least some of the time) to catch issues.

To generate a set of credentials, run [tiller_ssl.sh](docs/tiller-ssl/tiller_ssl.sh) from inside
`docs/tiller-ssl/`. This will create a CA cert, a cert/key pair for Tiller, and a client cert/key pair.
If debugging using the helm cli, include the tls flags. For example:
```bash
helm ls --all --tls-verify --tls-ca-cert docs/tiller-ssl/ca.cert.pem --tls-cert docs/tiller-ssl/tiller.cert.pem --tls-key docs/tiller-ssl/tiller.key.pem
```

See [Helm's tiller_ssl.md](https://github.com/helm/helm/blob/master/docs/tiller_ssl.md) for more details. 

### Charts
The Kibosh code loads charts from the `HELM_CHART_DIR`, which defaults to `charts`.
This directory can either be a single chart (with all the changes described in the
configuration, eg `plans.yaml` and `./plans`), or, directory where each
subdirectory is a chart. The multiple charts feature isn't yet supported by tile-generator.
```
charts
├── mariadb
│   ├── Chart.yaml
    ├── plans
    │   ├── medium.yaml
    │   └── small.yaml
    ├── plans.yaml
    ├── templates
...
└── mysql
    ├── Chart.yaml
    ├── plans
    │   └── default.yaml
...
```

We have modified [some example charts](https://github.com/cf-platform-eng/kibosh-sample/tree/master/sample-charts) from stable helm repository.
 
### Test
```bash
make test
```

To generate the test-doubles, after any interface change run: 
```bash
make generate
```

For manual testing, there is a [Python test harness](docs/dev-testing).

#### Integration Tests
The integration testing suite is located in [test/suite.py](test/suite.py). The suite will
sequentially run through the OSBAPI lifecycle and exit at the first failure (so as to leave things
in a state for debugging). [test/test_broker_base.py](test/test_broker_base.py) processes environment
variables `$BROKER_HOST`, `$BROKER_USERNAME`, `$BROKER_PASSWORD` to determine the broker to connect to. 


Prerequisites:
* Install required python packages listed in [test/requirements.txt](test/requirements.txt)     
    * i.e. `pip3 install -r test/requirements.txt`
* Unpackaged (untarred) MySQL helm chart is in `/charts` directory at root of kibosh project
* `kubectl` is available on the path and configured to talk to the same cluster that kibosh will provision to.

Run with command: `python3 test/suite.py`

## CI
* https://concourse.cfplatformeng.com/teams/main/pipelines/kibosh

The pipeline is backed by a cluster in the shared GKE account. The default admin
user in GKE has a password while Kibosh is configured to use a token. To create a user
in the cluster and fetch the token, do something like:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kibosh-concourse-ci
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: kibosh-concourse-ci
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: kibosh-concourse-ci
    namespace: kube-system
```

```bash
kubectl create -f [above contents in file].yml
kubectl get secrets --namespace=kube-system | grep "kibosh-concourse-ci"
kubectl get secret --namespace=kube-system kibosh-concourse-ci-token-pfnqs -o yaml
```

## Notes

Inline-style: 
![](docs/SeqDiagram.png)

<details>
  <summary>Sequence diagram source</summary>
  via https://www.websequencediagrams.com/ 

        title Kibosh

        operator->cf: deploy tile with kibosh and helm chart
        kibosh->cf: add offering to marketplaces via errand
        user->cf: cf create-service
        cf->kibosh: OSBAPI api provision call
        kibosh-> k8s: deploy chart
        user->cf: cf bind-service
        cf->kibosh: OSBAPI api bind call
        kibosh-> k8s: k8s api to get secrets & services
        k8s->kibosh: secrets and services
        kibosh->cf: secrets and services as credentials json
        cf->app: secrets and services as env vars
</details>

spec:
  hooks:
  - execContainer:
      command:
      - sh
      - -c
      - chroot /rootfs echo "Banner /etc/issue.net" >> /etc/ssh/sshd_config 
      image: busybox
  - execContainer:
      command:
      - sh
      - -c
      - chroot /rootfs apt-get update && chroot /rootfs apt-get install -y redis-server
      image: busybox
  fileAssets:
  - name: issues banner
    # Note if not path is specified the default path it /srv/kubernetes/assets/<name>
    path: /etc/issues.net
    roles: [Master,Node] # a list of roles to apply the asset to, zero defaults to all
    content: |
      This is a U.S. government service. Your use indicates your consent to monitoring, recording, and no expectation of privacy. Misuse is subject to criminal and civil penalties. VAN-TEST

  - name: sshd config
    # Note if not path is specified the default path it /srv/kubernetes/assets/<name>
    path: /etc/ssh/sshd_config
    roles: [Master,Node] # a list of roles to apply the asset to, zero defaults to all
    content: |
      Ciphers aes128-cbc,aes192-cbc,aes256-cbc,aes128-ctr,aes192-ctr,aes256-ctr,aes128-gcm@openssh.com,aes256-gcm@openssh.com
      GssapiAuthentication yes
      GssapiCleanUpCredentials yes
      HostKey /etc/ssh/ssh_host_rsa_key
      HostKey /etc/ssh/ssh_host_ecdsa_key
      HostKey /etc/ssh/ssh_host_ed25519_key
      MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com,hmac-sha2-512,hmac-sha2-256
      PasswordAuthentication no
      PermitRootLogin forced-commands-only
      Protocol 2
      RevokedKeys /etc/ssh/revoked_keys
      Subsystem sftp internal-sftp
      SyslogFacility AUTHPRIV
      TrustedUserCAKeys /etc/ssh/ca_keys
      UsePAM yes
      Banner /etc/issue.net





apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-support-van
  labels:
    k8s-app: kube-support-van
spec:
  selector:
    matchLabels:
      name: kube-support-van
  template:
    metadata:
      labels:
        name: kube-support-van
    spec:
      tolerations:
      - key: "node.kubernetes.io/disk-pressure"
        operator: "Exists"
        effect: "NoExecute"
        tolerationSeconds: 6000
      containers:
      - name: busybox
        image: busybox:1.29.2
        command: ["/bin/sh"]
        args: ["-c", "sleep 3600"]
        volumeMounts:
        - name: host
          mountPath: /host
      volumes:
      - name: host
        hostPath:
          path: /
          type: Directory
      nodeSelector:
        kubernetes.io/hostname: ip-10-9-34-38.us-gov-west-1.compute.internal




apiVersion: v1
kind: Pod
metadata:
 name: kube-support
 namespace: kube-system
spec:
  containers:
    - name: busybox
      image: busybox:1.29.2
      command: ["/bin/sh"]
      args: ["-c", "sleep 3600"]
      volumeMounts:
        - name: host
          mountPath: /host
  volumes:
    - name: host
      hostPath:
        path: /
        type: Directory
  nodeSelector:
    kubernetes.io/hostname: ip-10-9-34-38.us-gov-west-1.compute.internal
  tolerations:
    - key: "node.kubernetes.io/disk-pressure"
      operator: "Exists"
      effect: "NoExecute"
      tolerationSeconds: 6000