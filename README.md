## Setup k3s

```sh
    cd base-cluster
    mvn clean deploy
    export KUBECONFIG=/home/stefanw/git/wistefan/deployment-demo/base-cluster/target/k3s.yaml

    # enable storage
    kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.30/deploy/local-path-storage.yaml
```

## Deploy Trust Anchor

```sh
    helm repo add data-space-connector https://fiware.github.io/data-space-connector/
    cd trust-anchor
    helm template data-space-connector/trust-anchor --version 2.2.0 -f values.yaml --name-template=trust-anchor --namespace=trust-anchor --output-dir rendered
```

> :warning: Don´t forget to set the proper KUBECONFIG. You might deploy to an unwanted cluster elsewise.;)

Create namespace:
```sh
    kubectl create namespace trust-anchor
``` 

Use helm install:
```sh
    helm install trust-anchor data-space-connector/trust-anchor --version 0.2.0 -f values.yaml --namespace=trust-anchor
    watch kubectl get pods -n trust-anchor
```

Use kubectl apply:
```sh
    # if helm template was executed, this can be used to apply all files in the folder
    kubectl apply -R  -f rendered/    
    watch kubectl get pods -n trust-anchor
```

## Deploy consumer:

Create namespace:
```sh
    kubectl create namespace consumer
``` 

Create an identity for the consumer:

```shell
    mkdir consumer-identity

    # generate the private key - dont get confused about the curve, openssl uses the name `prime256v1` for `secp256r1`(as defined by P-256)
    openssl ecparam -name prime256v1 -genkey -noout -out consumer-identity/private-key.pem

    # generate corresponding public key
    openssl ec -in consumer-identity/private-key.pem -pubout -out consumer-identity/public-key.pem

    # create a (self-signed) certificate
    openssl req -new -x509 -key consumer-identity/private-key.pem -out consumer-identity/cert.pem -days 360

    # export the keystore
    openssl pkcs12 -export -inkey consumer-identity/private-key.pem -in consumer-identity/cert.pem -out consumer-identity/cert.pfx -name didPrivateKey

    # check the contents
    keytool -v -keystore consumer-identity/cert.pfx -list -alias didPrivateKey

    # generate did from the keystore
    wget https://github.com/wistefan/did-helper/releases/download/0.1.1/did-helper
    chmod +x did-helper
    ./did-helper -keystorePath ./consumer-identity/cert.pfx -keystorePassword=test
```

Deploy the key to the cluster

```shell
    kubectl create secret generic consumer-identity --from-file=consumer-identity/cert.pfx -n consumer
```

Use helm install:
```sh
    helm install consumer-dsc data-space-connector/data-space-connector --version 7.17.0 -f consumer/values.yaml --namespace=consumer
    watch kubectl get pods -n consumer
```

Access the issuer: http://keycloak-consumer.127.0.0.1.nip.io:8080/realms/test-realm/account/oid4vci

Register the consumer at the trust-anchor:

```shell
  curl -X POST http://til.127.0.0.1.nip.io:8080/issuer \
    --header 'Content-Type: application/json' \
    --data '{
      "did": "did:key:zDnaeu8wo7cYJyRUm15xoDuhEF2EB4x9Vph9AYph1Dg7JK8yW",
      "credentials": []
    }'
```

Get a list of the issuers:

```shell
    curl -X GET http://tir.127.0.0.1.nip.io:8080/v4/issuers
```
Decode at https://jwt.io/

### Verify its working

Get a credential from the consumer:
```shell
    export USER_CREDENTIAL=$(./scripts/get_credential_for_consumer.sh http://keycloak-consumer.127.0.0.1.nip.io:8080 operator-credential); echo ${USER_CREDENTIAL}
```

## Deploy provider

Create an identity for the provider:

```shell
    mkdir provider-identity

    # generate the private key - dont get confused about the curve, openssl uses the name `prime256v1` for `secp256r1`(as defined by P-256)
    openssl ecparam -name prime256v1 -genkey -noout -out provider-identity/private-key.pem

    # generate corresponding public key
    openssl ec -in provider-identity/private-key.pem -pubout -out provider-identity/public-key.pem

    # create a (self-signed) certificate
    openssl req -new -x509 -key provider-identity/private-key.pem -out provider-identity/cert.pem -days 360

    # export the keystore
    openssl pkcs12 -export -inkey provider-identity/private-key.pem -in provider-identity/cert.pem -out provider-identity/cert.pfx -name didPrivateKey

    # check the contents
    keytool -v -keystore provider-identity/cert.pfx -list -alias didPrivateKey

    # generate did from the keystore
    wget https://github.com/wistefan/did-helper/releases/download/0.1.1/did-helper
    chmod +x did-helper
    ./did-helper -keystorePath ./provider-identity/cert.pfx -keystorePassword=test
```

Create namespace:
```sh
    kubectl create namespace provider
```

Deploy the key to the cluster

```shell
    kubectl create secret generic provider-identity --from-file=provider-identity/cert.pfx -n provider
```

Use helm install:
```sh
    helm install provider-dsc data-space-connector/data-space-connector --version 7.17.0 -f provider/values.yaml --namespace=provider
    watch kubectl get pods -n provider
```

Register the provider at the trust-anchor:

```shell
  curl -X POST http://til.127.0.0.1.nip.io:8080/issuer \
    --header 'Content-Type: application/json' \
    --data '{
      "did": "did:key:zDnaenDzfrfaMSUPqZSrAcryFgZEEc6Hmg9VotkdREHd9tMoj",
      "credentials": []
    }'
```

### Verify its working

Get dataspace config:
```shell
    curl http://mp-data-service.127.0.0.1.nip.io:8080/.well-known/data-space-configuration
```

Get the openid-config:
```shell
    curl http://mp-data-service.127.0.0.1.nip.io:8080/.well-known/openid-configuration
```

Confifure the trusted-issuers-list:
```shell
    curl -X POST http://til-provider.127.0.0.1.nip.io:8080/issuer \
    --header 'Content-Type: application/json' \
    --data '{
        "did": "did:key:zDnaeu8wo7cYJyRUm15xoDuhEF2EB4x9Vph9AYph1Dg7JK8yW",
        "credentials": [
            {
                "credentialsType": "OperatorCredential"
            }
        ]
    }'
```


Unauthorized access not allowed:
```shell
    curl -s -X GET 'http://mp-data-service.127.0.0.1.nip.io:8080/ngsi-ld/v1/entities'
```

Prepare wallet-identity:
```shell
    mkdir wallet-identity
    chmod o+rw wallet-identity
    docker run -v $(pwd)/:/cert quay.io/wi_stefan/did-helper:0.1.1
```

Get an access token for the consumer:
```shell
    export ACCESS_TOKEN=$(./scripts/get_access_token_oid4vp.sh http://mp-data-service.127.0.0.1.nip.io:8080 $USER_CREDENTIAL operator); echo $ACCESS_TOKEN
```

Access the data-service:

```shell
    curl -s -X GET 'http://mp-data-service.127.0.0.1.nip.io:8080/ngsi-ld/v1/entities?type=EnergyReport' \
    --header 'Accept: application/json' \
    --header "Authorization: Bearer ${ACCESS_TOKEN}"
```

Add policy:

```shell
    curl -s -X 'POST' http://pap-provider.127.0.0.1.nip.io:8080/policy \
    -H 'Content-Type: application/json' \
    -d  '{ 
            "@context": {
              "dc": "http://purl.org/dc/elements/1.1/",
              "dct": "http://purl.org/dc/terms/",
              "owl": "http://www.w3.org/2002/07/owl#",
              "odrl": "http://www.w3.org/ns/odrl/2/",
              "rdfs": "http://www.w3.org/2000/01/rdf-schema#",
              "skos": "http://www.w3.org/2004/02/skos/core#"
            },
            "@id": "https://mp-operation.org/policy/common/type",
            "@type": "odrl:Policy",
            "odrl:permission": {
              "odrl:assigner": {
                "@id": "https://www.mp-operation.org/"
              },
              "odrl:target": {
                "@type": "odrl:AssetCollection",
                "odrl:source": "urn:asset",
                "odrl:refinement": [
                  {
                    "@type": "odrl:Constraint",
                    "odrl:leftOperand": "ngsi-ld:entityType",
                    "odrl:operator": {
                      "@id": "odrl:eq"
                    },
                    "odrl:rightOperand": "EnergyReport"
                  }
                ]
              },
              "odrl:assignee": {
                "@id": "vc:any"
              },
              "odrl:action": {
                "@id": "odrl:read"
              }
            }
          }'
```