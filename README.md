# Accessing IBM MQ queue manager running on CP4I (OpenShift)

>NOTE: This is a work in progress. It's not finished yet.

## Prepare a queue manager on OpenShift

### Prepare certificates

Create a separate directory for the certificates
```sh
mkdir workdir
cd workdir
```

If you don't have the CA certificate create a self-signed one:
```sh
openssl genpkey -algorithm rsa -pkeyopt rsa_keygen_bits:4096 -out ca.key
openssl req -x509 -new -nodes -key ca.key -sha512 -days 365 -subj "/CN=example-selfsigned-ca" -out ca.crt
```

Create key and certificate signing request for queue manager:
```sh
openssl req -new -nodes -out queuemanager.csr -newkey rsa:4096 -keyout queuemanager.key -subj '/CN=queuemanager'
```

Create queue manager certificate signed with CA:
```sh
openssl x509 -req -in queuemanager.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out queuemanager.crt -days 365 -sha512
```

### Create secret with queue manager certificates

So far, we have:
- ca.crt
- queuemanager.crt
- queuemanager.key

Create secret in the OpenShift namespace where the queue manager instance will be created:
```sh
oc create secret generic queuemanager --type="kubernetes.io/tls" --from-file=tls.key=queuemanager.key --from-file=tls.crt=queuemanager.crt --from-file=ca.crt
```

### Prepare queue manager configuration

This is an example of the ConfigMap with the MQ minimal configuration. In the following examples, we will create variations of it with different configurations.

Create the ConfigMap in the same namespace where the queue manager instance will be running.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: queuemanager-configmap
data:
  queuemanager.mqsc: |
    ALTER AUTHINFO(SYSTEM.DEFAULT.AUTHINFO.IDPWOS)  AUTHTYPE(IDPWOS) CHCKCLNT(NONE) CHCKLOCL(NONE)
    REFRESH SECURITY TYPE(CONNAUTH)

    * More MQSC commands...

  queuemanager.ini: |
    Service:
        Name=AuthorizationService
        EntryPoints=14
        SecurityPolicy=UserExternal
```

### Create queue manager

Note how the previously created ConfigMap and TLS secret are referenced. 

Please note also that the namespace is *qmtest* - change it to your namespace. Please accept license before applying YAML by changing *spec.license.accept* from *false* to *true*.

```yaml
apiVersion: mq.ibm.com/v1beta1
kind: QueueManager
metadata:
  name: qm2
  namespace: qmtest
spec:
  license:
    accept: false
    license: L-CYPF-CRPF3H
    use: NonProduction
  queueManager:
    name: QM2
    resources:
      limits:
        cpu: 500m
      requests:
        cpu: 500m
    mqsc:
    - configMap:
        name: queuemanager-configmap
        items:
        - queuemanager.mqsc
    ini:
    - configMap:
        name: queuemanager-configmap
        items:
        - queuemanager.ini
    storage:
      queueManager:
        type: ephemeral
    # availability:
    #   type: NativeHA
  version:  9.4.3.0-r2
  web:
    console:
      authentication:
        provider: integration-keycloak
      authorization:
        provider: integration-keycloak
    enabled: true
  pki:
    keys:
      - name: default
        secret:
          secretName: queuemanager
          items:
            - tls.key
            - tls.crt
            - ca.crt
```

## EXAMPLE 1: Testing QM to QM connection using Podman