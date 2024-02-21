---
title: "Managing access"
excerpt: "Managing access for users and applications."
categories: security
slug: managing-access
toc: true
---

You can secure your {{site.data.reuse.es_name}} resources by managing the access each user and application has to each resource.

An {{site.data.reuse.es_name}} cluster can be configured to expose any number of internal or external Kafka listeners. These listeners provide the mechanism for Kafka client applications to communicate with the Kafka brokers. The bootstrap address is used for the initial connection to the cluster. The address will resolve to one of the brokers in the cluster and respond with metadata describing all the relevant connection information for the remaining brokers.

Each Kafka listener providing a connection to {{site.data.reuse.es_name}} can be configured to authenticate connections with Mutual TLS, SCRAM-SHA-512, or OAuth authentication mechanisms.

Additionally, the {{site.data.reuse.es_name}} cluster can be configured to authorize operations sent via an authenticated listener.

**Note:** Schemas in the Apicurio Registry in {{site.data.reuse.es_name}} are a special case and are secured using the resource type of `topic` combined with a prefix of `__schema_`. You can control the ability of users and applications to create, delete, read, and update schemas.

## Accessing the {{site.data.reuse.es_name}} UI and CLI

When Kafka authentication is enabled, the {{site.data.reuse.es_name}} UI and CLI requires you to log in by using an {{site.data.reuse.icpfs}} Identity and Access Management (IAM) user, or by using a Kafka user configured with SCRAM-SHA-512 authentication, depending on the authentication type set for the `adminUI` in the `EventStreams` custom resource (see [configuring UI and CLI security](../../installing/configuring/#configuring-ui-and-cli-security)).

{{site.data.reuse.iam_note}} For other Kubernetes platforms, you can log in by using a Kafka user configured with SCRAM-SHA-512 authentication. You can create a Kafka user by applying a [`KafkaUser` custom resource](#creating-a-kafkauser-by-using-yaml).



### Managing access to the UI and CLI with IAM

Access for groups and users is managed through IAM teams. If you have not previously created any teams, the administrative user credentials can be used to [set up a team](https://www.ibm.com/docs/en/cloud-paks/foundational-services/3.23?topic=users-managing-teams){:target="_blank"}.

Access to the {{site.data.reuse.es_name}} UI and CLI requires an IAM user with a role of `Cluster Administrator`, `CloudPakAdministrator`,  or `Administrator`. The role can be set for the user, or for the group the user is part of.

The default cluster administrator (admin) IAM user credentials are stored in a secret within the `ibm-common-services` namespace. To retrieve the username and password:

1. {{site.data.reuse.openshift_cli_login}}
2. Extract and decode the current {{site.data.reuse.icpfs}} admin username:

   ```shell
   oc --namespace ibm-common-services get secret platform-auth-idp-credentials -o jsonpath='{.data.admin_username}' | base64 --decode
   ```

3. Extract and decode the current {{site.data.reuse.icpfs}} admin password:
  
   ```shell
   oc --namespace ibm-common-services get secret platform-auth-idp-credentials -o jsonpath='{.data.admin_password}' | base64 --decode
   ```

**Note:** The password is auto-generated in accordance with the default password rules for {{site.data.reuse.icpfs}}. To change the username or password of the admin user, see the instructions about [changing the cluster administrator access credentials](https://www.ibm.com/docs/en/cloud-paks/foundational-services/3.23?topic=configurations-changing-cloud-pak-administrator-access-credentials#administrator-pwd){:target="_blank"}.

The admin user has the `Cluster Administrator` role granting full access to all resources within the cluster, including {{site.data.reuse.es_name}}.


Any groups or users added to an IAM team with the `Cluster Administrator` role can log in to the {{site.data.reuse.es_name}} UI and CLI. Any groups or users with the `Administrator` role will not be able to log in until the namespace that contains the {{site.data.reuse.es_name}} cluster is [added as a resource](https://www.ibm.com/docs/en/cloud-paks/foundational-services/3.23?topic=teams-add-resources-team){:target="_blank"} for the IAM team.

If Kafka authorization is enabled by setting `spec.strimziOverrides.kafka.authorization` to `type: runas`, operations by IAM users are automatically mapped to a Kafka principal with authorization for the required Kafka resources.

**Note:** If you are using OAuth authorization in Kafka, ensure you add all admin user IDs to the `superUsers` property in the Kafka authorization configuration within the `EventStreams` custom resource, as described in [enabling OAuth authorization](../../installing/configuring/#enable-oauth-authorization).

### Managing access to the UI and CLI with SCRAM

When access to the {{site.data.reuse.es_name}} UI and CLI is set up with [SCRAM authentication](../../installing/configuring/#configuring-ui-and-cli-security), any Kafka users configured to use SCRAM-SHA-512 are able to log in to the UI and run CLI commands by using the Kafka user name and the SCRAM password, which is contained in the secret generated by the creation of the Kafka user (as described in [retrieving credentials](#retrieving-the-credentials-of-a-kafkauser)).

When a SCRAM user logs in to the UI or runs CLI commands, the permissions of the `KafkaUser` determine which panels or functionality are available to the user. This allows the cluster administrator to grant different permissions to different users, permitting administrative actions through the UI or CLI based on the role of a specific user.

The following table describes the UI panels and the permissions required to access them.

| UI Panel        | Permissions     | Additional Information |
|:----------------|:----------------|:---------------------------|
| **Topics**          | `topic.read` or `topic.describe` | - If the user also has either `topic.create` or `cluster.create` permissions, the **Topic Create** button is enabled. <br> - If the user has `topic.delete` permission, the **Topic Delete** button in the overflow menu is enabled. |
| **Topic Producer**  |                  | This panel is disabled for SCRAM authentication. |
| **Schema registry** | `schema.read`    | If the user also has the `schema.alter` permission, then the **Add Schema** button is enabled. |
| **Metrics**         |                  | This panel is disabled for SCRAM authentication. |
| **Consumer groups** | `group.read`     |                    |
| **Geo-replication**  | `cluster.alter` | Geo-replication is enabled for SCRAM authentication. |
| **Generate credentials**  | `cluster.alter` | ![Event Streams 11.2.4 icon]({{ 'images' | relative_url }}/11.2.4.svg "In Event Streams 11.2.4 and later.") **Generate credentials** buttons are enabled in the **Cluster connection** panel. |
| **Starter application** | `topic.read` and `topic.write` | When generating the starter application, the current user ID and password will be configured in the properties. `topic.create` permission is required to create new topics within the Starter App wizard. |

The following tables describe the CLI commands and the permissions required to run them.

| Topic CLI command     | Permissions required |
|:----------------------|:---------------------|
| `kubectl es topics` | `topic.read` or `topic.describe` |
| `kubectl es topic` |  `topic.read` and `topic.describeConfigs` |
| `kubectl es topic-create` | `topic.create`     |
| `kubectl es topic-partitions-set` | `topic.alter` and `topic.describeConfigs` |
| `kubectl es topic-update` | `topic.alterConfigs` and `topic.describe` |
| `kubectl es topic-delete` <br/>`kubectl es topic-delete-records` | `topic.delete` |

| Consumer group CLI command | Permissions required |
|:---------------------------|:---------------------|
| `kubectl es groups`       | `group.read`      |
| `kubectl es group`        | `group.read` and `topic.describe`  |
| `kubectl es group-reset`  | `group.read` and `topic.read`  |
| `kubectl es group-delete` | `group.delete` |

| Schema CLI command | Permissions required |
|:-------------------|:---------------------|
| `kubectl es schemas` <br/>`kubectl es schema` <br/>`kubectl es schema-export` | `schema.read` |
| `kubectl es schema-add` <br/>`kubectl es schema-verify` <br/>`kubectl es schema-modify` | `schema.read` and `schema.alter` |
| `kubectl es schema-remove`  <br/>`kubectl es schema-import` | `schema.alter` |

| Geo-replication CLI command | Permissions required |
|:----------------------------|:---------------------|
| `kubectl es geo-cluster` <br/>`kubectl es geo-cluster-add` <br/>`kubectl es geo-cluster-connect` <br/>`kubectl es geo-cluster-remove` <br/>`kubectl es geo-clusters` <br/>`kubectl es geo-replicator-create` <br/>`kubectl es geo-replicator-delete` <br/>`kubectl es geo-replicator-pause` <br/>`kubectl es geo-replicator-restart` <br/>`kubectl es geo-replicator-resume` <br/>`kubectl es geo-replicator-topics-add` <br/>`kubectl es geo-replicator-topics-remove` | `cluster.alter` |

| Broker or cluster CLI command | Permissions required |
|:------------------------------|:---------------------|
| `kubectl es acls` | `cluster.describe` |
| `kubectl es broker` | `cluster.describe` and `cluster.describeConfigs` |
| `kubectl es broker-config` <br/>`kubectl es cluster` <br/>`kubectl es cluster-config` | `cluster.describeConfigs` |
| `kubectl es certificates`| No permissions required |

{{site.data.reuse.es_cli_kafkauser_note}}
![Event Streams 11.2.4 icon]({{ 'images' | relative_url }}/11.2.4.svg "In Event Streams 11.2.4 and later.") You can also use the {{site.data.reuse.es_name}} UI to generate the Kafka users.

The following table describes the mapping of these permissions to the Kafka user ACL definitions.

| Permissions        | ACL Resource Type | ACL Operation | ACL Pattern Type | ACL Resource Name |
|:-------------------|-------------------|---------------|------------------|-------------------|
| `cluster.alter`    | cluster           | ALTER         | N/A              | N/A               |
| `cluster.create`   | cluster           | CREATE        | N/A              | N/A               |
| `cluster.describe` | cluster           | DESCRIBE      | N/A              | N/A               |
| `cluster.describeConfigs` | cluster    | DESCRIBE_CONFIGS | N/A              | N/A               |
| `group.read`  [^1]    | group             | READ          | literal/prefix   | '*', \<group prefix\> or \<group name\> |
| `group.delete`  [^1]  | group             | DELETE          | literal/prefix   | '*', \<group prefix\> or \<group name\> |
| `schema.alter`  [^2]  | topic             | ALTER         | prefix           | '\__schema\_'     |
| `schema.read`  [^2]   | topic             | READ          | prefix           | '\__schema\_'     |
| `topic.create` [^1]   | topic             | CREATE        | literal/prefix   | '*', \<topic prefix\> or \<topic name\> |
|  `topic.delete` [^1]  | topic  | DELETE | literal/prefix | '*', \<topic prefix\> or \<topic name\> |
| `topic.describe` [^1] | topic             | DESCRIBE      | literal/prefix   | '*', \<topic prefix\> or \<topic name\> |
| `topic.read` [^1]     | topic             | READ          | literal/prefix   | '*', \<topic prefix\> or \<topic name\> |
| `topic.write` [^1]    | topic             | WRITE         | literal/prefix   | '*', \<topic prefix\> or \<topic name\> |
| `topic.describeConfigs` [^1] | topic      | DESCRIBE_CONFIGS | literal/prefix   | '*', \<topic prefix\> or \<topic name\> |

[^1]: Topics and groups can be configured with an asterisk (*), meaning all topics and groups, a specific prefix name, or an exact name. These will influence which resources are listed within the panels, and they also determine whether certain actions are permitted, for example, whether a user can create a topic.

[^2]: {{site.data.reuse.es_name}} Schema ACLs are topic resource ACLs, but with a specific prefix, `__schema_`. Schema permissions do not target a set of named schemas through prefixed names or a single schema through an explicitly defined name. Only define ACLs for schemas with the exact settings defined in the table earlier, permitting the operation against all schemas (otherwise, parts of the UI might not work as expected).

For more information about ACLs, see the [authorization section](#authorization).

## Managing access to Kafka resources

Each Kafka listener exposing an authenticated connection to {{site.data.reuse.es_name}} requires credentials to be presented when connecting. SCRAM-SHA-512 and Mutual TLS authentication credentials are created by using a `KafkaUser` custom resource, where the `spec.authentication.type` field has a value that matches the Kafka listener authentication type.

OAuth authentication does not require any `KafkaUser` custom resources to be created.

You can create a `KafkaUser` by using the {{site.data.reuse.es_name}} UI or CLI. It is also possible to create a `KafkaUser` by using the {{site.data.reuse.openshift_short}} UI or CLI, or the underlying Kubernetes API by applying a `KafkaUser` operand request.

**Note:** You must have authenticated to the {{site.data.reuse.openshift_short}} UI with an IAM user to be able to generate Kafka users within the UI. This capability is not available when authenticating with a SCRAM user ID.

To assist in generating compatible `KafkaUser` credentials, the {{site.data.reuse.es_name}} UI indicates which authentication mechanism is being configured for each Kafka listener.

**Warning:** Do not use or modify the internal {{site.data.reuse.es_name}} `KafkaUsers` named `<cluster>-ibm-es-kafka-user`, `<cluster>-ibm-es-georep-source-user`, `<cluster>-ibm-es-iam-admin-kafka-user`, or `<cluster>-ibm-es-ac-reg`. These are reserved to be used internally within the {{site.data.reuse.es_name}} instance.

### Creating a `KafkaUser` in the {{site.data.reuse.es_name}} UI

**Note:** In {{site.data.reuse.es_name}} 11.2.3 and earlier versions you must have authenticated with an IAM user to be able to generate Kafka users within the {{site.data.reuse.es_name}} UI.

1. {{site.data.reuse.es_ui_login}}
2. Click the **Connect to this cluster** tile to view the **Cluster connection** panel.
3. Go to the **Kafka listener and Credentials** section.
4. To generate credentials for external clients, click **External**, or to generate credentials for internal clients, click **Internal**.
5. Click the **Generate SCRAM credential** or **Generate TLS credential** button next to the required listener to view the credential generation dialog.
6. Follow the instructions to generate credentials with desired permissions.\\
**Note:** If your cluster does not have authorization enabled, the permission choices will not have any effect.

The generated credential appears after the listener bootstrap address:
* For SCRAM credentials, two tabs are displayed: **Username and password** and **Basic Authentication**. The SCRAM username and password combination is used by Kafka applications, while the Basic Authentication credential is for use as an HTTP authorization header.
* For TLS credentials, a download button is displayed, providing a zip archive with the required certificates and keys.

A `KafkaUser` will be created with the entered credential name.

The cluster truststore is not part of the above credentials ZIP file archive. This certificate is required for all external connections, and is available to download from the **Cluster connection** panel under the **Certificates** heading within the {{site.data.reuse.openshift_short}} UI, or by running the command `kubectl es certificates`. Upon downloading the PKCS12 certificate, the certificate password will also be displayed.

Additionally, to retrieve endpoint details, view the `EventStreams` custom resource in the OpenShift web console as follows:

1. {{site.data.reuse.openshift_ui_login}}
2. Expand **Operators** in the navigation on the left, and click **Installed Operators**.
3. Locate the operator that manages your {{site.data.reuse.es_name}} instance in the namespace. It is called **{{site.data.reuse.es_name}}** in the **NAME** column.
4. Click the **{{site.data.reuse.es_name}}** link in the row and click the **{{site.data.reuse.es_name}}** tab. This lists the **{{site.data.reuse.es_name}}** instances related to this operator.
5. Find your instance in the **Name** column and click the link for the instance.
6. Go to the menu of your {{site.data.reuse.es_name}} instance and select **Edit EventStreams**.
7. In the YAML view, scroll down to the `status.endpoints` section.
8. The admin API URL is the value in the `uri` property of the `admin` section.
9. The schema registry URL is the value in the `uri` property of the `apicurioregistry` section.

![How to retrieve endpoints]({{ 'images/' | relative_url }}/endpoints.png "Screen capture showing where to retrieve the endpoints from."){:height="50%" width="50%"}

### Creating a `KafkaUser` in the {{site.data.reuse.es_name}} CLI

{{site.data.reuse.es_cli_kafkauser_note}}
![Event Streams 11.2.4 icon]({{ 'images' | relative_url }}/11.2.4.svg "In Event Streams 11.2.4 and later.") You can also use the {{site.data.reuse.es_name}} UI to generate the Kafka users.

1. {{site.data.reuse.cp_cli_login}}
2. {{site.data.reuse.es_cli_init_111}}
3. Use the `kafka-user-create` command to create a `KafkaUser` with the accompanying permissions.\\
**Note:** If your cluster does not have authorization enabled, the permission choices will not have any effect.

For example, to create SCRAM credentials with authorization to create topics and schemas, and also to produce to (including transactions) and consume from every topic:

```shell
cloudctl es kafka-user-create \
  --name my-user \
  --consumer \
  --producer \
  --schema-topic-create \
  --all-topics \
  --all-groups \
  --all-txnids \
  --auth-type scram-sha-512
```

For information about all options provided by the command, use the `--help` flag:\\
`cloudctl es kafka-user-create --help`



### UI and CLI `KafkaUser` authorization

`KafkaUsers` created by using the UI and CLI can only be configured with permissions for the most common operations against Kafka resources. You can later modify the created `KafkaUser` to [add additional ACL rules.](#authorization)

#### Creating a `KafkaUser` in the {{site.data.reuse.openshift_short}} UI

**Note:** You must have authenticated to the {{site.data.reuse.openshift_short}} UI with an IAM user to be able to generate Kafka users within the UI.

Navigate to the **Event Streams** installed operator menu and select the **KafkaUser** tab. Click `Create KafkaUser`. The YAML view contains sample `KafkaUser` definitions to consume, produce, or modify every resource.

###  Creating a `KafkaUser` by using YAML

You can create a Kafka user by creating a `KafkaUser` custom resource in a YAML file, and then running `kubectl apply -f <filename>.yaml`.

For example, the following YAML creates SCRAM credentials for a Kafka user with authorization to create topics, groups, and schemas, and also to produce to and consume from every topic. It also provides administrator access to the {{site.data.reuse.es_name}} UI by generating a Kubernetes `Secret` containing the Base64-encoded SCRAM password for the `scram-sha-512` authentication type. You can use the password to [access the {{site.data.reuse.es_name}} UI](#managing-access-to-the-ui-with-scram).

```yaml
apiVersion: eventstreams.ibm.com/v1beta2
kind: KafkaUser
metadata:
  name: <username>
  namespace: <namespace>
  labels:
    eventstreams.ibm.com/cluster: <cluster-name>
spec:
  authentication:
    type: scram-sha-512
  authorization:
    type: simple
    acls:
      - resource:
          type: cluster
        operations: 
          - Alter
      - resource:
          type: topic
          name: '*'
          patternType: literal
        operations: 
          - Create
          - Delete
          - Read
          - Write
      - resource:
          type: topic
          name: __schema_
          patternType: prefix
        operations: 
          - Alter
          - Read
      - resource:
          type: group
          name: '*'
          patternType: literal
        operations:
          - Read
```

### Retrieving the credentials of a `KafkaUser`

When a `KafkaUser` custom resource is created, the Entity Operator within {{site.data.reuse.es_name}} will create the principal in ZooKeeper with appropriate ACL entries. It will also create a Kubernetes `Secret` that contains the Base64-encoded SCRAM password for the `scram-sha-512` authentication type, or the Base64-encoded certificates and keys for the `tls` authentication type.

You can retrieve the credentials by using the name of the `KafkaUser`. For example, you can retrieve the credentials as follows:

1. Set the current namespace:

   ```shell
   kubectl config set-context --current --namespace=<namespace>
   ```


2. {{site.data.reuse.cncf_cli_login}}
3. Use the following command to retrieve the required `KafkaUser` data, adding the `KafkaUser` name and your chosen namespace: \\

   ```shell
   kubectl get kafkauser <name> -o jsonpath='{"username: "}{.status.username}{"\nsecret-name: "}{.status.secret}{"\n"}'
   ```

   The command provides the following output:
   - The principal username
   - The name of the Kubernetes `Secret`, which includes the namespace, containing the SCRAM password or the TLS certificates.

4. Decode the credentials.

    - For SCRAM, use the `secret-name` from the previous step to retrieve the password and decode it:

      ```shell
      kubectl get secret <secret-name> -o jsonpath='{.data.password}' | base64 --decode
      ```

    - For TLS, get the credentials, decode them, and write each certificates and keys to files:

      ```shell
      kubectl get secret <secret-name> -o jsonpath='{.data.ca\.crt}' | base64 --decode > ca.crt
      ```

      ```shell
      kubectl get secret <secret-name> -o jsonpath='{.data.user\.crt}' | base64 --decode > user.crt
      ```

      ```shell
      kubectl get secret <secret-name> -o jsonpath='{.data.user\.key}' | base64 --decode > user.key
      ```

      ```shell
      kubectl get secret <secret-name> -o jsonpath='{.data.user\.p12}' | base64 --decode > user.p12
      ```

      ```shell
      kubectl get secret <secret-name> -o jsonpath='{.data.user\.password}' | base64 --decode
      ```

5. Additionally, you can also retrieve endpoint details such as the admin API URL and the schema registry URL by using the following commands:

  - To retrieve the admin API URL from the `status.endpoints` list in the `EventStreams` custom resource, run the following command:

    ```shell
    kubectl get eventstreams <instance-name> -n <namespace> -o jsonpath="{.status.endpoints[?(@.name=='admin')].uri}"
    ```

  - To retrieve the schema registry URL from the `status.endpoints` list in the `EventStreams` custom resource, run the following command:

    ```shell
    kubectl get eventstreams <instance-name> -n <namespace> -o jsonpath="{.status.endpoints[?(@.name=='apicurioregistry')].uri}"
    ```

The cluster truststore certificate is required for all external connections and is available to download from the **Cluster connection** panel under the **Certificates** heading, within the {{site.data.reuse.openshift_short}} UI, or by running the command `kubectl es certificates`. Upon downloading the PKCS12 certificate, the certificate password will also be displayed.

Similarly, if you are using {{site.data.reuse.openshift_short}}, you can inspect these `KafkaUser` and `Secret` resources by using the web console.

**Warning:** Do not use or modify the internal {{site.data.reuse.es_name}} `KafkaUsers` named `<cluster>-ibm-es-kafka-user`, `<cluster>-ibm-es-georep-source-user`, `<cluster>-ibm-es-iam-admin-kafka-user`, or `<cluster>-ibm-es-ac-reg`. These are reserved to be used internally within the {{site.data.reuse.es_name}} instance.

## Authorization

### What resource types can I secure?

Within {{site.data.reuse.es_name}}, you can secure access to the following resource types, where the names in parentheses are the resource type names used in Access Control List (ACL) rules:
* Topics (`topic`): you can control the ability of users and applications to create, delete, read, and write to a topic.
* Consumer groups (`group`): you can control an application's ability to join a consumer group.
* Transactional IDs (`transactionalId`): you can control the ability to use the transaction capability in Kafka.
* Cluster (`cluster`): you can control operations that affect the whole cluster, including idempotent writes.

**Note:** Schemas in the Apicurio Registry in {{site.data.reuse.es_name}} are a special case and are secured using the resource type of `topic` combined with a prefix of `__schema_`. You can control the ability of users and applications to create, delete, read, and update schemas.

### What are the available Kafka operations?

Access control in Apache Kafka is defined in terms of operations and resources. Operations are actions performed on a resource, and each operation maps to one or more APIs or requests.

| Resource type   | Operation       | Kafka API                  |
|:----------------|:----------------|:---------------------------|
| topic           | Alter           | CreatePartitions           |
|                 | AlterConfigs    | AlterConfigs               |
|                 |                 | IncrementalAlterConfigs    |
|                 | Create          | CreateTopics               |
|                 |                 | Metadata                   |
|                 | Delete          | DeleteRecords              |
|                 |                 | DeleteTopics               |
|                 | Describe        | ListOffsets                |
|                 |                 | Metadata                   |
|                 |                 | OffsetFetch                |
|                 |                 | OffsetForLeaderEpoch       |
|                 | DescribeConfigs | DescribeConfigs            |
|                 | Read            | Fetch                      |
|                 |                 | OffsetCommit               |
|                 |                 | TxnOffsetCommit            |
|                 | Write           | AddPartitionsToTxn         |
|                 |                 | Produce                    |
|                 | All             | All topic APIs             |
| &nbsp;          | &nbsp;          | &nbsp;                     |
| group           | Delete          | DeleteGroups               |
|                 | Describe        | DescribeGroup              |
|                 |                 | FindCoordinator            |
|                 |                 | ListGroups                 |
|                 | Read            | AddOffsetsToTxn            |
|                 |                 | Heartbeat                  |
|                 |                 | JoinGroup                  |
|                 |                 | LeaveGroup                 |
|                 |                 | OffsetCommit               |
|                 |                 | OffsetFetch                |
|                 |                 | SyncGroup                  |
|                 |                 | TxnOffsetCommit            |
|                 | All             | All group APIs             |
| &nbsp;          | &nbsp;          | &nbsp;                     |
| transactionalId | Describe        | FindCoordinator            |
|                 | Write           | AddOffsetsToTxn            |
|                 |                 | AddPartitionsToTxn         |
|                 |                 | EndTxn                     |
|                 |                 | InitProducerId             |
|                 |                 | Produce                    |
|                 |                 | TxnOffsetCommit            |
|                 | All             | All txnid APIs             |
| &nbsp;          | &nbsp;          | &nbsp;                     |
| cluster         | Alter           | AlterReplicaLogDirs        |
|                 |                 | CreateAcls                 |
|                 |                 | DeleteAcls                 |
|                 | AlterConfigs    | AlterConfigs               |
|                 |                 | IncrementalAlterConfigs    |
|                 | ClusterAction   | ControlledShutdown         |
|                 |                 | ElectPreferredLeaders      |
|                 |                 | Fetch                      |
|                 |                 | LeaderAndISR               |
|                 |                 | OffsetForLeaderEpoch       |
|                 |                 | StopReplica                |
|                 |                 | UpdateMetadata             |
|                 |                 | WriteTxnMarkers            |
|                 | Create          | CreateTopics               |
|                 |                 | Metadata                   |
|                 | Describe        | DescribeAcls               |
|                 |                 | DescribeLogDirs            |
|                 |                 | ListGroups                 |
|                 |                 | ListPartitionReassignments |
|                 | DescribeConfigs | DescribeConfigs            |
|                 | IdempotentWrite | InitProducerId             |
|                 |                 | Produce                    |
|                 | All             | All cluster APIs           |

### Implicitly-derived operations

Certain operations provide additional implicit operation access to applications.

When granted `Read`, `Write`, or `Delete`, applications implicitly derive the `Describe` operation.
When granted `AlterConfigs`, applications implicitly derive the `DescribeConfigs` operation.

For example, to `Produce` to a topic, the `Write` operation for the `topic` resource is required, which will implicitly derive the `Describe` operation required to get the topic metadata.

### Access Control List (ACL) rules

Access to resources is assigned to applications through an Access Control List (ACL), which consists of rules. An ACL rule includes an operation on a resource together with the additional fields listed in the following tables. A `KafkaUser` custom resource contains the binding of an ACL to a principal, which is an entity that can be authenticated by the {{site.data.reuse.es_name}} instance.

An ACL rule adheres to the following schema:

| Property     | Type   | Description                                                                                  |
|:-------------|:-------|:---------------------------------------------------------------------------------------------|
| `host`       | string | The host from which the action described in the ACL rule is allowed.                         |
| `operations` | array  | An array of strings that lists all the operations which will be allowed on the chosen resource. |
| `resource`   | object | Indicates the resource for which the ACL rule applies.                                       |

The resource objects used in ACL rules adhere to the following schema:

| Property      | Type   | Description |
|:--------------|:-------|:------------|
| `type`        | string | Can be one of `cluster`, `group`, `topic` or `transactionalId`. |
| `name`        | string | Identifies the value that the ACL rule will authenticate against when receiving incoming requests, rejecting anything that does not match. For example, the `topic` that can be accessed based on the topic name, or the `transactionalId` that can be used by a client. The `name` value can be used in combination with the `patternType` value to use the prefix pattern. |
| `patternType` | string | Describes the pattern used in the resource field. The supported types are `literal` and `prefix`. With literal pattern type, the resource field will be used as a definition of a full topic name. With prefix pattern type, the resource name will be used only as a prefix. <br> The default value is `literal`. |

Using the information about schemas and resource-operations described in the previous tables, the `spec.authorization.acls` list for a `KafkaUser` can be created as follows:

```yaml
# ...
spec:
# ...
  authorization:
    # ...
    acls:
      - host: '*'
        resource:
          type: topic
          name: 'client-'
          patternType: prefix
        operations: 
          - Write
```

In this example, an application using this `KafkaUser` would be allowed to write to any topic beginning with **client-** (for example, **client-records** or **client-billing**) from any host machine.

**Note:** The write operation also implicitly derives the required describe operation that Kafka clients require to understand the data model of a topic.

The following is an example ACL rule that provides access to read Schemas:

```yaml
- host: '*'
  resource:
    type: topic
    name: '__schema_'
    patternType: prefix
  operations:
    - Read
```

### Using OAuth

Open Authorization (OAuth) is an open standard for authorization that allows client applications secure delegated access to specified resources. OAuth works over HTTPS and uses access tokens for authorization rather than credentials.

Ensure you have enabled OAuth in your {{site.data.reuse.es_name}} [installation](../../installing/configuring/#enabling-oauth).

You can [configure OAuth authentication](../../installing/configuring/#enable-oauth-authentication) by using either fast JWT token authentication or token introspection.

**Note:** You cannot configure OAuth authentication with the {{site.data.reuse.es_name}} [REST producer API](../../connecting/rest-api/) or [Schema registry](../../schemas/setting-java-apps/#preparing-the-setup). Both endpoints can only be configured with SCRAM-SHA-512 or Mutual TLS authentication.

You can also configure [OAuth authorization](../../installing/configuring/#enable-oauth-authorization).

## Revoking access for an application

As each application will be using credentials provided through a `KafkaUser` instance, deleting the instance will revoke all access for that application. Individual ACL rules can be deleted from a `KafkaUser` instance to remove the associated control on a resource operation.


#### Footnotes