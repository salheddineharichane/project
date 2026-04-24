OpenTelemetry Gateway → Kafka : un topic par namespace
Pour créer un topic Kafka dynamique basé sur le namespace, tu peux utiliser le Kafka Exporter avec un template dans le champ topic qui référence l'attribut de ressource k8s.namespace.name.
Configuration de la Gateway (OpenTelemetryCollector)
yamlapiVersion: opentelemetry.io/v1beta1
kind: OpenTelemetryCollector
metadata:
  name: otel-gateway
spec:
  mode: deployment
  config:
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318

    processors:
      batch:
        timeout: 5s
        send_batch_size: 1000

      # Assure-toi que l'attribut namespace est bien dans les resource attributes
      resource:
        attributes:
          - key: k8s.namespace.name
            from_attribute: k8s.namespace.name
            action: upsert

    exporters:
      kafka:
        brokers:
          - kafka-broker:9092
        topic: "otel-logs-${k8s.namespace.name}"   # ← template dynamique
        topic_from_attribute: "k8s.namespace.name"  # ← attribut source
        encoding: otlp_proto
        logs:
          topic: "otel-logs-${k8s.namespace.name}"
        producer:
          max_message_bytes: 1000000
          required_acks: 1
          compression: snappy
        # Création automatique des topics
        metadata:
          full: true
        auth:
          tls:
            insecure: true  # à adapter selon ta config

    service:
      pipelines:
        logs:
          receivers: [otlp]
          processors: [resource, batch]
          exporters: [kafka]
Le point clé : topic_from_attribute
Le champ topic_from_attribute indique au Kafka Exporter d'utiliser la valeur d'un attribut de ressource comme nom de topic :
yamlexporters:
  kafka:
    brokers:
      - kafka-broker:9092
    topic: "otel-logs-default"           # topic fallback si attribut absent
    topic_from_attribute: "k8s.namespace.name"
    encoding: otlp_proto
Résultat : les logs du namespace production iront dans production, ceux de staging dans staging, etc.

Vérifier que l'attribut est bien propagé depuis le DaemonSet
Dans la config de ton DaemonSet, assure-toi que le processor k8sattributes extrait bien le namespace :
yamlprocessors:
  k8sattributes:
    auth_type: serviceAccount
    extract:
      metadata:
        - k8s.namespace.name   # ← indispensable
        - k8s.pod.name
        - k8s.node.name
    pod_association:
      - sources:
          - from: resource_attribute
            name: k8s.pod.ip
      - sources:
          - from: connection

Création automatique des topics sur Kafka
Si Kafka n'a pas auto.create.topics.enable=true, tu dois soit :

L'activer côté Kafka (simple pour un test) :

yaml# Dans ta config Kafka broker
auto.create.topics.enable: "true"

Pré-créer les topics via un Job OCP :

bashkafka-topics.sh --create \
  --bootstrap-server kafka:9092 \
  --topic production \
  --partitions 3 \
  --replication-factor 2

Debug rapide
Pour vérifier que l'attribut k8s.namespace.name est bien présent dans les logs reçus par la gateway :
yamlexporters:
  debug:
    verbosity: detailed

service:
  pipelines:
    logs:
      receivers: [otlp]
      processors: [resource, batch]
      exporters: [kafka, debug]   # ajoute debug temporairement
Tu veux que je t'aide aussi à configurer les ACLs Kafka par topic ou la partie RBAC OCP pour le service account du collector ?
