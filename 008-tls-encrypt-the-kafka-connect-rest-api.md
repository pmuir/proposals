# TLS encrypting the Kafka Connect REST API

This proposal reduces access to the Kafka Connect REST API by enabling TLS encryption on the Kafka Connect REST listener when the connectors running on the Kafka Connect cluster are managed by the `KafkaConnector` or `KafkaMirrroMaker2` operators.

## Current situation

Currently, instances of Kafka Connect that are deployed by the Strimzi operators are configured with the default REST API endpoint settings.
This means that the Kafka Connect REST API endpoint uses HTTP on port 8083 and that the Strimzi `KafkaConnect`, `KafkaConnector` and `KafkaMirrorMaker2` operators make unsecured REST client calls based on this default configuration.

The default network policies created by the Strimzi operators restrict incoming REST API calls to only allow access from the operator pod.
Users can define further network policies can be provided to override the default policy and allow wider access.

## Motivation

There is currently no way to TLS encrypt the Kafka Connect REST API, leaving the Connect REST listener as perhaps the last key endpoint that cannot currently be restricted using Strimzi.

## Proposal

This proposal changes the behaviour of the `KafkaMirrorMaker2` operator and the `KafkaConnect` operator.
In the `KafkaConnect` operator case, the behaviour is only changed when the `strimzi.io/use-connector-resources: "true"` annotation is applied in the `KafkaConnect` CR (meaning that all connectors running on the Connect cluster are managed by the `KafkaConnector` operator).

The default behaviour of the `KafkaMirrorMaker2` operator and the `KafkaConnect` operator will be to deploy Kafka Connect clusters with a single TLS encrypted REST API listener.
A self-signed TLS certificate and key required to enable TLS encryption on the REST API listener are generated by the operator, stored in secrets for use by the operator clients and the Kafka Connect pods.
The certificates and keys are mounted in each Kafka Connect pod, which create an SSL truststore and keystore from the certificate and key and set the following properties in the generated connect configuration to enable an HTTPS connection:

```
listeners: https://:8443
rest.advertised.listener: https
rest.advertised.port: 8443
listeners.https.ssl.client.auth: none
listeners.https.ssl.truststore.location: /tmp/kafka/kafka-connect-rest.truststore.p12
listeners.https.ssl.truststore.password: ***generated password***
listeners.https.ssl.truststore.type: PKCS12
listeners.https.ssl.keystore.location: /tmp/kafka/kafka-connect-rest.keystore.p12
listeners.https.ssl.keystore.password: ***generated password***
listeners.https.ssl.keystore.type: PKCS12
```

To communicate with the encrypted Connect REST API listener, the `KafkaMirrorMaker2`, `KafkaConnect` and `KafkaConnector` operators require modifications to the `AbstractConnectOperator` to apply the TLS certs as truststore options on the Vertx WebClient.

### Enabling a plain HTTP REST API listener

A new field is added to the `KafkaConnect` and `KafkaMirrorMaker2` specs named `restListeners`, which allows users to configure an additional unencrypted HTTP listener on port 8083 in the Kafka Connect configuration.
This enables users to access the REST API without TLS if required.
The `KafkaConnector`, `KafkaConnect` and `KafkaMirrorMaker2` operators will continue to use the TLS encrypted listener when an unencrypted HTTP listener is enabled.

The `restListeners` field is designed for potential future extension, and is a list of items that contain a `protocol` field.
For this proposal, the `protocol` field only supports the value `http`. One (or more) items in the `restListeners` list will add the unencrypted HTTP listener on port 8083.

For example, the following CR will enable the HTTP listener on port 8083:
```
apiVersion: kafka.strimzi.io/v1beta1
kind: KafkaConnect
spec:
  restListeners:
  - protocol: http
  ...
```

The schema for `restListeners` is as follows:
```
openAPIV3Schema:
  type: object
  properties:
    spec:
      type: object
      properties:
        restListeners:
          type: array
          items:
            type: object
            properties:
              protocol:
                type: string
                enum:
                - http
                description: The protocol of the REST listener.
            required:
            - protocol
          description: List of additional REST listeners.
        ...
```


### Certificate renewals

The self-signed TLS certificate will have a 1-year expiration.
Thirty days before the self-signed TLS certificate expires, the operator will automatically renew the certificate, replace the old self-signed certificate, and restart the pods in a controlled manner to ensure the new certificates are in use by Kafka Connect without losing availability.

This would need to be a multi-phase process consisting of the following steps:

1. Generate new certificate
2. Distribute the new certificate to all pods and cluster operator truststores
3. Replace the key and roll all Connect pods
4. When the old certificate expires, remove it from the truststores and roll all Connect pods


## Compatibility

Switching on TLS encryption for the Connect REST API does not directly affect users of Strimzi custom resources, but given that the Connect REST API is a well known Kafka interface, there may be Strimzi users who currently access the REST API directly, for example with tooling or for monitoring.
TLS encryption would mean users could no longer access the REST API as they did before. However, this can be overridden using the `spec.restListeners` field if desired.


## Rejected Alternatives

### Annotation alternative to enable a plain HTTP REST API listener

Alternatively, a new annotation `strimzi.io/enable-plain-rest-listener` could be added, to avoid changing the spec.
Setting the `strimzi.io/enable-plain-rest-listener` annotation value to `true` would add the additional unencrypted HTTP listener on port 8083.
This was rejected as it is generally better to have configuration in the `spec`, and the proposed `restListeners` structure could be extended in future enhancements.
