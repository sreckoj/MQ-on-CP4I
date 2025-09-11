# Accessing IBM MQ queue manager running on CP4I (OpenShift)

>NOTE: This is a work in progress. It's not finished yet.

## Prepare certificates

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


