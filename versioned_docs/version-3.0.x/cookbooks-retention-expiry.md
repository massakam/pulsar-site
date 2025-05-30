---
id: cookbooks-retention-expiry
title: Message retention and expiry
sidebar_label: "Message retention and expiry"
---

````mdx-code-block
import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';
````


Pulsar brokers are responsible for handling messages that pass through Pulsar, including [persistent storage](concepts-architecture-overview.md#persistent-storage) of messages. By default, for each topic, brokers only retain messages that are in at least one backlog. A backlog is a set of unacknowledged messages for a particular subscription. As a topic can have multiple subscriptions, a topic can have multiple backlogs.

As a consequence, no messages are retained (by default) on a topic that has not had any subscriptions created for it.

(Note that messages that are no longer being stored are not necessarily immediately deleted, and may still be accessible until the next ledger rollover. Because clients cannot predict when rollovers may happen, it is not wise to rely on a rollover not happening at an inconvenient point in time.)

In Pulsar, you can modify this behavior, with namespace granularity, in two ways:

* You can persistently store messages that are not within a backlog (because they've been acknowledged by every existing subscription, or because there are no subscriptions) by setting [retention policies](#retention-policies).
* Messages that are not acknowledged within a specified timeframe can be automatically acknowledged, by specifying the [time to live](#time-to-live-ttl) (TTL).

Pulsar's [admin interface](admin-api-overview.md) enables you to manage both retention policies and TTL with namespace granularity (and thus within a specific tenant and either on a specific cluster or in the [`global`](concepts-architecture-overview.md#clusters) cluster).


> #### Retention and TTL solve two different problems
> * Message retention: Keep the data for at least X hours (even if acknowledged)
> * Time-to-live: Discard data after some time (by automatically acknowledging)
>
> Most applications will want to use at most one of these.


## Retention policies

By default, when a Pulsar message arrives at a broker, the message is stored until it has been acknowledged on all subscriptions, at which point it is marked for deletion. You can override this behavior and retain messages that have already been acknowledged on all subscriptions by setting a *retention policy* for all topics in a given namespace. Retention is based on both a *size limit* and a *time limit*.

The diagram below illustrates the concept of message retention.
![](/assets/retention-new.svg)

Retention policies are useful when you use the Reader interface. The Reader interface does not use acknowledgments, and messages do not exist within backlogs. It is required to configure retention for Reader-only use cases.

When you set a retention policy on topics in a namespace, you must set **both** a *size limit* (via `defaultRetentionSizeInMB`) and a *time limit* (via `defaultRetentionTimeInMinutes`) . You can refer to the following table to set retention policies in `pulsar-admin` and Java.

|Time limit|Size limit| Message retention      |
|----------|----------|------------------------|
| -1       | -1       | Infinite retention  |
| -1       | >0       | Based on the size limit  |
| >0       | -1       | Based on the time limit  |
| 0        | 0        | Disable message retention, which means messages are not reserved (by default) |
| 0        | >0       | Invalid  |
| >0       | 0        | Invalid  |
| >0       | >0       | Acknowledged messages or messages with no active subscription will not be retained when either time or size reaches the limit. |

The retention settings apply to all messages on topics that do not have any subscriptions, or to messages that have been acknowledged by all subscriptions. The retention policy settings do not affect unacknowledged messages on topics with subscriptions. The unacknowledged messages are controlled by the backlog quota.

When a retention limit on a topic is exceeded, the oldest message is marked for deletion until the set of retained messages falls within the specified limits again.

### Defaults

You can set message retention at instance level with the following two parameters: `defaultRetentionTimeInMinutes` and `defaultRetentionSizeInMB`. By default, both parameters are set to `0`, which means disabling message retention.

For more information on the two parameters, refer to the [`broker.conf`](reference-configuration.md#broker) configuration file.

### Set retention policy

You can set a retention policy for a namespace by specifying the namespace, a size limit and a time limit in `pulsar-admin`, REST API and Java. And all the non-partitioned topics and topic partitions will apply the specified data retention policy for the namespace unless you overwrite the data retention policy for the specific topic by using topic level policies.

````mdx-code-block
<Tabs groupId="api-choice"
  defaultValue="pulsar-admin"
  values={[{"label":"pulsar-admin","value":"pulsar-admin"},{"label":"REST API","value":"REST API"},{"label":"Java","value":"Java"}]}>
<TabItem value="pulsar-admin">

You can use the [`set-retention`](pathname:///reference/#/@pulsar:version_reference@/pulsar-admin/namespaces?id=set-retention) subcommand and specify a namespace, a size limit using the `-s`/`--size` flag, and a time limit using the `-t`/`--time` flag.

In the following example, the size limit is set to 10 GB and the time limit is set to 3 hours for each topic within the `my-tenant/my-ns` namespace.
- When the size of messages reaches 10 GB on a topic within 3 hours, the acknowledged messages will not be retained.
- After 3 hours, even if the message size is less than 10 GB, the acknowledged messages will not be retained.

```shell
pulsar-admin namespaces set-retention my-tenant/my-ns \
--size 10G \
--time 3h
```

In the following example, the time is not limited and the size limit is set to 1 TB. The size limit determines the retention.

```shell
pulsar-admin namespaces set-retention my-tenant/my-ns \
--size 1T \
--time -1
```

In the following example, the size is not limited and the time limit is set to 3 hours. The time limit determines the retention.

```shell
pulsar-admin namespaces set-retention my-tenant/my-ns \
--size -1 \
--time 3h
```

To achieve infinite retention, set both values to `-1`.

```shell
pulsar-admin namespaces set-retention my-tenant/my-ns \
--size -1 \
--time -1
```

To disable the retention policy, set both values to `0`.

```shell
pulsar-admin namespaces set-retention my-tenant/my-ns \
--size 0 \
--time 0
```

</TabItem>
<TabItem value="REST API">

[](swagger:/admin/v2/Namespaces_setRetention)

:::note

To disable the retention policy, you need to set both the size and time limit to `0`. Set either size or time limit to `0` is invalid.

:::

</TabItem>
<TabItem value="Java">

```java
int retentionTime = 10; // 10 minutes
int retentionSize = 500; // 500 megabytes
RetentionPolicies policies = new RetentionPolicies(retentionTime, retentionSize);
admin.namespaces().setRetention(namespace, policies);
```

</TabItem>

</Tabs>
````

### Get retention policy

You can fetch the retention policy for a namespace by specifying the namespace. The output will be a JSON object with two keys: `retentionTimeInMinutes` and `retentionSizeInMB`.

````mdx-code-block
<Tabs groupId="api-choice"
  defaultValue="pulsar-admin"
  values={[{"label":"pulsar-admin","value":"pulsar-admin"},{"label":"REST API","value":"REST API"},{"label":"Java","value":"Java"}]}>
<TabItem value="pulsar-admin">

Use the [`get-retention`](pathname:///reference/#/@pulsar:version_reference@/pulsar-admin/namespaces?id=get-retention) subcommand and specify the namespace.

##### Example

```shell
pulsar-admin namespaces get-retention my-tenant/my-ns
{
  "retentionTimeInMinutes": 10,
  "retentionSizeInMB": 500
}
```

</TabItem>
<TabItem value="REST API">

[](swagger:/admin/v2/Namespaces_getRetention)

</TabItem>
<TabItem value="Java">

```java
admin.namespaces().getRetention(namespace);
```

</TabItem>

</Tabs>
````

## Backlog quotas

*Backlogs* are sets of unacknowledged messages for a topic that have been stored by bookies. Pulsar stores all unacknowledged messages in backlogs until they are processed and acknowledged.

You can control the allowable size and/or time of backlogs, at the namespace level, using *backlog quotas*. Pulsar uses a quota to enforce a hard limit on the logical size of the backlogs in a topic. Backlog quota triggers an alert policy (for example, producer exception) once the quota limit is reached.

The diagram below illustrates the concept of backlog quota.
![](/assets/backlog-quota-new.svg)

Setting a backlog quota involves setting:

* an allowable *size and/or time threshold* for each topic in the namespace
* a *retention policy* that determines which action the [broker](reference-terminology.md#broker) takes if the threshold is exceeded.

The following retention policies are available:

Policy | Action
:------|:------
`producer_request_hold` | The producer holds the message and retris until client configuration `sendTimeoutMs` is exceeded.
`producer_exception` | The producer throws an exception when trying to send a message
`consumer_backlog_eviction` | The broker will begin discarding backlog messages

:::note
The scope of retention includes the backlog, so retention must be greater than the backlog quota. Otherwise, Pulsar will throw an error when both are set.
:::

Backlog quotas are handled at the namespace level. They can be managed via:

### Set size/time thresholds and backlog retention policies

You can set a size and/or time threshold and backlog retention policy for all of the topics in a [namespace](reference-terminology.md#namespace) by specifying the namespace, a size limit and/or a time limit in second, and a policy by name.

````mdx-code-block
<Tabs groupId="api-choice"
  defaultValue="pulsar-admin"
  values={[{"label":"pulsar-admin","value":"pulsar-admin"},{"label":"REST API","value":"REST API"},{"label":"Java","value":"Java"}]}>
<TabItem value="pulsar-admin">

Use the [`set-backlog-quota`](pathname:///reference/#/@pulsar:version_reference@/pulsar-admin/namespaces?id=set-backlog-quota) subcommand and specify a namespace, a size limit using the `-l`/`--limit` , `-lt`/`--limitTime` flag to limit backlog, a retention policy using the `-p`/`--policy` flag and a policy type using `-t`/`--type` (default is destination_storage).

##### Example

```shell
pulsar-admin namespaces set-backlog-quota my-tenant/my-ns \
--limit 2G \
--policy producer_request_hold
```

```shell
pulsar-admin namespaces set-backlog-quota my-tenant/my-ns/my-topic \
--limitTime 3600 \
--policy producer_request_hold \
--type message_age
```

</TabItem>
<TabItem value="REST API">

[](swagger:/admin/v2/Namespaces_getBacklogQuotaMap)

</TabItem>
<TabItem value="Java">

```java
admin.namespaces().setBacklogQuota(namespace, BacklogQuota.builder()
        .retentionPolicy(RetentionPolicy.producer_request_hold)
        .limitSize(2147483648L)
        .limitTime(60 * 60)
        .build());
```

</TabItem>

</Tabs>
````

### Get backlog threshold and backlog retention policy

You can see which size threshold and backlog retention policy has been applied to a namespace.

````mdx-code-block
<Tabs groupId="api-choice"
  defaultValue="pulsar-admin"
  values={[{"label":"pulsar-admin","value":"pulsar-admin"},{"label":"REST API","value":"REST API"},{"label":"Java","value":"Java"}]}>
<TabItem value="pulsar-admin">

Use the [`get-backlog-quotas`](pathname:///reference/#/@pulsar:version_reference@/pulsar-admin/namespaces?id=get-backlog-quotas) subcommand and specify a namespace. Here's an example:

```shell
pulsar-admin namespaces get-backlog-quotas my-tenant/my-ns
{
  "destination_storage": {
    "limit" : 2147483648,
    "policy" : "producer_request_hold"
  }
}
```

</TabItem>
<TabItem value="REST API">

[](swagger:/admin/v2/Namespaces_getBacklogQuotaMap)

</TabItem>
<TabItem value="Java">

```java
Map<BacklogQuota.BacklogQuotaType,BacklogQuota> quotas =
  admin.namespaces().getBacklogQuotas(namespace);
```

</TabItem>

</Tabs>
````

### Remove backlog quotas

````mdx-code-block
<Tabs groupId="api-choice"
  defaultValue="pulsar-admin"
  values={[{"label":"pulsar-admin","value":"pulsar-admin"},{"label":"REST API","value":"REST API"},{"label":"Java","value":"Java"}]}>
<TabItem value="pulsar-admin">

Use the [`remove-backlog-quota`](pathname:///reference/#/@pulsar:version_reference@/pulsar-admin/namespaces?id=remove-backlog-quota) subcommand and specify a namespace, use `t`/`--type` to specify backlog type to remove(default is destination_storage). Here's an example:

```shell
pulsar-admin namespaces remove-backlog-quota my-tenant/my-ns
```

</TabItem>
<TabItem value="REST API">

[](swagger:/admin/v2/Namespaces_removeBacklogQuota)

</TabItem>
<TabItem value="Java">

```java
admin.namespaces().removeBacklogQuota(namespace);
```

</TabItem>

</Tabs>
````

### Clear backlog

#### pulsar-admin

Use the [`clear-backlog`](pathname:///reference/#/@pulsar:version_reference@/pulsar-admin/namespaces?id=clear-backlog) subcommand.

##### Example

```shell
pulsar-admin namespaces clear-backlog my-tenant/my-ns
```

By default, you will be prompted to ensure that you want to clear the backlog for the namespace. You can override the prompt using the `-f`/`--force` flag.

## Time to live (TTL)

By default, Pulsar stores all unacknowledged messages forever. This can lead to heavy disk space usage in cases where a lot of messages are going unacknowledged. If disk space is a concern, you can set a time to live (TTL) that determines how long unacknowledged messages will be retained.

The TTL parameter is like a stopwatch attached to each message that defines the amount of time a message is allowed to stay unacknowledged. When the TTL expires, Pulsar automatically moves the message to the acknowledged state (and thus makes it ready for deletion).

The diagram below illustrates the concept of TTL.
![](/assets/ttl.svg)

### Set the TTL for a namespace

````mdx-code-block
<Tabs groupId="api-choice"
  defaultValue="pulsar-admin"
  values={[{"label":"pulsar-admin","value":"pulsar-admin"},{"label":"REST API","value":"REST API"},{"label":"Java","value":"Java"}]}>
<TabItem value="pulsar-admin">

Use the [`set-message-ttl`](pathname:///reference/#/@pulsar:version_reference@/pulsar-admin/namespaces?id=set-message-ttl) subcommand and specify a namespace and a TTL (in seconds) using the `-ttl`/`--messageTTL` flag.

##### Example

```shell
pulsar-admin namespaces set-message-ttl my-tenant/my-ns \
--messageTTL 120 # TTL of 2 minutes
```

</TabItem>
<TabItem value="REST API">

[](swagger:/admin/v2/Namespaces_setNamespaceMessageTTL)

</TabItem>
<TabItem value="Java">

```java
admin.namespaces().setNamespaceMessageTTL(namespace, ttlInSeconds);
```

</TabItem>

</Tabs>
````

### Get the TTL configuration for a namespace

````mdx-code-block
<Tabs groupId="api-choice"
  defaultValue="pulsar-admin"
  values={[{"label":"pulsar-admin","value":"pulsar-admin"},{"label":"REST API","value":"REST API"},{"label":"Java","value":"Java"}]}>
<TabItem value="pulsar-admin">

Use the [`get-message-ttl`](pathname:///reference/#/@pulsar:version_reference@/pulsar-admin/namespaces?id=get-message-ttl) subcommand and specify a namespace.

##### Example

```shell
pulsar-admin namespaces get-message-ttl my-tenant/my-ns
60
```

</TabItem>
<TabItem value="REST API">

[](swagger:/admin/v2/Namespaces_getNamespaceMessageTTL)

</TabItem>
<TabItem value="Java">

```java
admin.namespaces().getNamespaceMessageTTL(namespace)
```

</TabItem>

</Tabs>
````

### Remove the TTL configuration for a namespace

````mdx-code-block
<Tabs groupId="api-choice"
  defaultValue="pulsar-admin"
  values={[{"label":"pulsar-admin","value":"pulsar-admin"},{"label":"REST API","value":"REST API"},{"label":"Java","value":"Java"}]}>
<TabItem value="pulsar-admin">

Use the [`remove-message-ttl`](pathname:///reference/#/@pulsar:version_reference@/pulsar-admin/namespaces?id=remove-message-ttl) subcommand and specify a namespace.

##### Example

```shell
pulsar-admin namespaces remove-message-ttl my-tenant/my-ns
```

</TabItem>
<TabItem value="REST API">

[](swagger:/admin/v2/Namespaces_removeNamespaceMessageTTL)

</TabItem>
<TabItem value="Java">

```java
admin.namespaces().removeNamespaceMessageTTL(namespace)
```

</TabItem>

</Tabs>
````

## Delete messages from namespaces

When it comes to the physical storage size, message expiry and retention are just like two sides of the same coin.
* The backlog quota and TTL parameters prevent disk size from growing indefinitely, as Pulsar's default behavior is to persist unacknowledged messages.
* The retention policy allocates storage space to accommodate the messages that are supposed to be deleted by Pulsar by default.

In conclusion, the size of your physical storage should accommodate the sum of the backlog quota and the retention size.

The message deletion rate (releasing rate of disk space) can be determined by multiple factors.

- **Segment rollover period**: basically, the segment rollover period is how often a new segment is created. Once a new segment is created, the old segment will be deleted. By default, this happens either when you have written 50,000 entries (messages) or have waited 240 minutes. You can tune this in your broker.

- **Entry log rollover period**: multiple ledgers in BookKeeper are interleaved into an [entry log](https://bookkeeper.apache.org/docs/4.11.1/getting-started/concepts/#entry-logs). In order for a ledger that has been deleted, the entry log must all be rolled over.
The entry log rollover period is configurable but is purely based on the entry log size. For details, see [here](https://bookkeeper.apache.org/docs/4.11.1/reference/config/#entry-log-settings). Once the entry log is rolled over, the entry log can be garbage collected.

- **Garbage collection interval**: because entry logs have interleaved ledgers, to free up space, the entry logs need to be rewritten. The garbage collection interval is how often BookKeeper performs garbage collection. which is related to minor compaction and major compaction of entry logs. For details, see [here](https://bookkeeper.apache.org/docs/4.11.1/reference/config/#entry-log-compaction-settings).

The diagram below illustrates one of the cases that the consumed storage size is larger than the given limits for backlog and retention. Messages over the retention limit are kept because other messages in the same segment are still within retention period.
![](/assets/retention-storage-size.svg)

If you do not have any retention period and you never have much of a backlog, the upper limit for retained messages, which are acknowledged, equals the Pulsar segment rollover period + entry log rollover period + (garbage collection interval * garbage collection ratios).
