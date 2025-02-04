# Steps to Create Cluster on Platform External


## Prerequisites

- Install hypershift cli
- Install hypershift operator

## Create a Hosted Cluster

```sh
cat <<'EOF' > ./myvars
export CLUSTERS_NAMESPACE="clusters"
export HOSTED="external-aws"
export HOSTED_CLUSTER_NS="cluster-${HOSTED}"

#export PULL_SECRET_FILE=/path/to/your-pull-secret
export PULL_SECRET_NAME="${HOSTED}-pull-secret"
export PULL_SECRET_CONTENT=$(cat $PULL_SECRET_FILE)

export MACHINE_CIDR="10.0.0.0/16"
export OCP_RELEASE_VERSION="4.19.0-ec.1"
export OCP_ARCH="x86_64"
export BASEDOMAIN="devcluster.openshift.com"

export SSH_PUB=$(cat ~/.ssh/id_rsa.pub)
EOF
source ./myvars


envsubst <<"EOF" | oc apply -f -
apiVersion: v1
kind: Namespace
metadata:
 name: ${CLUSTERS_NAMESPACE}
EOF

export PS64=$(echo -n ${PULL_SECRET_CONTENT} | base64 -w0)
envsubst <<"EOF" | oc apply -f -
apiVersion: v1
data:
 .dockerconfigjson: ${PS64}
kind: Secret
metadata:
 name: ${PULL_SECRET_NAME}
 namespace: ${CLUSTERS_NAMESPACE}
type: kubernetes.io/dockerconfigjson
EOF

envsubst <<"EOF" | oc apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: ${HOSTED}-ssh-key
  namespace: ${CLUSTERS_NAMESPACE}
stringData:
  id_rsa.pub: ${SSH_PUB}
EOF


envsubst <<"EOF" | oc apply -f -
apiVersion: hypershift.openshift.io/v1beta1
kind: HostedCluster
metadata:
  name: ${HOSTED}
  namespace: ${CLUSTERS_NAMESPACE}
spec:
  release:
    image: "quay.io/openshift-release-dev/ocp-release:${OCP_RELEASE_VERSION}-${OCP_ARCH}"
  pullSecret:
    name: ${PULL_SECRET_NAME}
  sshKey:
    name: "${HOSTED}-ssh-key"
  networking:
    serviceCIDR: "172.31.0.0/16"
    podCIDR: "10.132.0.0/14"
    machineCIDR: "${MACHINE_CIDR}"
  platform:
    type: External
  infraID: ${HOSTED}
  dns:
    baseDomain: ${BASEDOMAIN}
  services:
  - service: APIServer
    servicePublishingStrategy:
      type: LoadBalancer
  - service: Ignition
    servicePublishingStrategy:
      type: LoadBalancer
EOF

envsubst <<"EOF" | oc apply -f -
apiVersion: hypershift.openshift.io/v1alpha1
kind: NodePool
metadata:
  name: ${HOSTED}-workers
  namespace: ${CLUSTERS_NAMESPACE}
spec:
  clusterName: ${HOSTED}
  replicas: 0
  management:
    autoRepair: false
    upgradeType: Replace
  platform:
    type: None
  release:
    image: "quay.io/openshift-release-dev/ocp-release:${OCP_RELEASE_VERSION}-${OCP_ARCH}"
EOF
```
