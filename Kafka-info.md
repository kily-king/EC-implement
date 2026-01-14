Below is an **implementation-ready section** you can append directly after the *Kafka Topic Definitions* in your architecture or platform doc.
It includes **Terraform (IaC-first)** and **Kafka CLI (ops / validation)** definitions.

---

# Kafka Topic Provisioning – Terraform & CLI

## 1. Provisioning Approach

Kafka topics are provisioned as part of the **core data platform infrastructure** and must exist **before** application producers, Kafka MirrorMaker, or downstream consumers are deployed.

Two supported provisioning methods are defined:

* **Terraform** – primary, repeatable, environment-safe
* **Kafka CLI** – operational validation and emergency use only

---

## 2. Terraform Definitions (Recommended)

### 2.1 Terraform Provider (Kafka)

```hcl
provider "kafka" {
  bootstrap_servers = var.kafka_bootstrap_servers
  tls_enabled       = true
}
```

---

### 2.2 Common Topic Configuration (Reusable)

```hcl
locals {
  default_topic_config = {
    "cleanup.policy"        = "delete"
    "retention.ms"          = "604800000" # 7 days
    "compression.type"      = "producer"
    "min.insync.replicas"   = "2"
  }
}
```

---

### 2.3 Kafka Topic Resources

#### Telemetry Events

```hcl
resource "kafka_topic" "device_events" {
  name               = "device-events"
  partitions         = 12
  replication_factor = 3
  config             = local.default_topic_config
}
```

#### Heartbeat Events

```hcl
resource "kafka_topic" "device_heartbeats" {
  name               = "device-heartbeats"
  partitions         = 6
  replication_factor = 3
  config             = local.default_topic_config
}
```

#### Fault Events

```hcl
resource "kafka_topic" "device_faults" {
  name               = "device-faults"
  partitions         = 6
  replication_factor = 3
  config             = local.default_topic_config
}
```

#### Transaction Events

```hcl
resource "kafka_topic" "device_transactions" {
  name               = "device-transactions"
  partitions         = 12
  replication_factor = 3
  config             = local.default_topic_config
}
```

---

### 2.4 Input Variables

```hcl
variable "kafka_bootstrap_servers" {
  description = "Kafka bootstrap servers"
  type        = list(string)
}
```

---

### 2.5 Outputs (Optional but Recommended)

```hcl
output "kafka_topics" {
  value = [
    kafka_topic.device_events.name,
    kafka_topic.device_heartbeats.name,
    kafka_topic.device_faults.name,
    kafka_topic.device_transactions.name
  ]
}
```

---

## 3. Kafka CLI Definitions (Operational Use)

> ⚠️ CLI usage is **not** the source of truth.
> It is intended for validation, troubleshooting, or non-prod recovery.

### 3.1 Topic Creation

```bash
kafka-topics.sh --bootstrap-server <broker:9092> \
  --create \
  --topic device-events \
  --partitions 12 \
  --replication-factor 3 \
  --config retention.ms=604800000 \
  --config min.insync.replicas=2
```

```bash
kafka-topics.sh --bootstrap-server <broker:9092> \
  --create \
  --topic device-heartbeats \
  --partitions 6 \
  --replication-factor 3 \
  --config retention.ms=604800000 \
  --config min.insync.replicas=2
```

```bash
kafka-topics.sh --bootstrap-server <broker:9092> \
  --create \
  --topic device-faults \
  --partitions 6 \
  --replication-factor 3 \
  --config retention.ms=604800000 \
  --config min.insync.replicas=2
```

```bash
kafka-topics.sh --bootstrap-server <broker:9092> \
  --create \
  --topic device-transactions \
  --partitions 12 \
  --replication-factor 3 \
  --config retention.ms=604800000 \
  --config min.insync.replicas=2
```

---

### 3.2 Topic Validation

```bash
kafka-topics.sh --bootstrap-server <broker:9092> --describe --topic device-events
```

Validate:

* Partition count
* Replication factor
* ISR count

---

### 3.3 Increase Partitions (Safe Operation)

```bash
kafka-topics.sh --bootstrap-server <broker:9092> \
  --alter \
  --topic device-events \
  --partitions 18
```

📌 Partition decreases are **not supported**.

---

## 4. Security & Governance Notes

* Topic creation restricted to platform-admin role
* Producers granted **Write** access only
* Consumers granted **Read** access only
* MirrorMaker requires **topic auto-create disabled** and explicit topic existence

---

## 5. Operational Definition of Done

* Topics created via Terraform
* `kafka-topics.sh --describe` confirms configuration
* MirrorMaker consumes without `UNKNOWN_TOPIC_OR_PARTITION` errors
* Parallel consumer groups verified

---

### 🔜 Optional Extensions

I can next:

* Add **ACL definitions (Terraform + CLI)**
* Add **Amazon MSK–specific Terraform**
* Convert this into **Sprint tasks with estimates**

Just tell me which one to do next.
