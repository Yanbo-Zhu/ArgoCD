
# 1 总览
Cluster credentials are stored in secrets same as repositories or repository credentials. Each secret must have label `argocd.argoproj.io/secret-type: cluster`.

The secret data must include following fields:

- `name` - cluster name
- `server` - cluster api server url
- `namespaces` - optional comma-separated list of namespaces which are accessible in that cluster. Cluster level resources would be ignored if namespace list is not empty.
- `clusterResources` - optional boolean string (`"true"` or `"false"`) determining whether Argo CD can manage cluster-level resources on this cluster. This setting is used only if the list of managed namespaces is not empty.
- `project` - optional string to designate this as a project-scoped cluster.
- `config` - JSON representation of following data structure:


```
# Basic authentication settings
username: string
password: string
# Bearer authentication settings
bearerToken: string
# IAM authentication configuration
awsAuthConfig:
    clusterName: string
    roleARN: string
    profile: string
# Configure external command to supply client credentials
# See https://godoc.org/k8s.io/client-go/tools/clientcmd/api#ExecConfig
execProviderConfig:
    command: string
    args: [
      string
    ]
    env: {
      key: value
    }
    apiVersion: string
    installHint: string
# Transport layer security configuration settings
tlsClientConfig:
    # Base64 encoded PEM-encoded bytes (typically read from a client certificate file).
    caData: string
    # Base64 encoded PEM-encoded bytes (typically read from a client certificate file).
    certData: string
    # Server should be accessed without verifying the TLS certificate
    insecure: boolean
    # Base64 encoded PEM-encoded bytes (typically read from a client certificate key file).
    keyData: string
    # ServerName is passed to the server for SNI and is used in the client to check server
    # certificates against. If ServerName is empty, the hostname used to contact the
    # server is used.
    serverName: string

```


# 2 EKS


EKS cluster secret example using argocd-k8s-auth and IRSA:

```
apiVersion: v1
kind: Secret
metadata:
  name: mycluster-secret
  labels:
    argocd.argoproj.io/secret-type: cluster
type: Opaque
stringData:
  name: "mycluster.example.com"
  server: "https://mycluster.example.com"
  config: |
    {
      "awsAuthConfig": {
        "clusterName": "my-eks-cluster-name",
        "roleARN": "arn:aws:iam::<AWS_ACCOUNT_ID>:role/<IAM_ROLE_NAME>"
      },
      "tlsClientConfig": {
        "insecure": false,
        "caData": "<base64 encoded certificate>"
      }        
    }

```










