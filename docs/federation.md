---
title: Federation Plugin
---
<!--
Copyright (c) 2005-2024 Broadcom. All Rights Reserved. The term "Broadcom" refers to Broadcom Inc. and/or its subsidiaries.

All rights reserved. This program and the accompanying materials
are made available under the terms of the under the Apache License,
Version 2.0 (the "License”); you may not use this file except in compliance
with the License. You may obtain a copy of the License at

https://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# Exchange and Queue Federation

## Overview {#overview}

The high-level goal of the Federation plugin is to transmit messages between brokers without
requiring clustering. This is useful for a number of reasons.

### Loose Coupling of Nodes or Clusters

The federation plugin can transmit messages between brokers
(or clusters) in different administrative domains:

 * they may be hosted in different data centers, potentially on different continents
 * they may have different users, virtual hosts, permissions and purpose
 * they may run on different versions of RabbitMQ and Erlang
 * they may be of different sizes
 * they may be

### WAN friendliness

The federation plugin communication is entirely asynchronous and assumes that connections between
clusters will fail from time to time. So it tolerates intermittent connectivity
well and does not create coupling between remote clusters (in terms of availability).

### Specificity

A broker can contain federated _and_ local-only components to best
fit the desired architecture of the system.

### Scalability with Growing Connected Node Count

Federation does not require O(n<sup>2</sup>) connections between
_N_ brokers (although this is the easiest way to set things up).


## What Does It Do? {#what-does-it-do}

The federation plugin make it possible to _federate_ exchanges and queues.
A federated exchange or queue can receive
messages from one or more remote clusters called _upstreams_ (to be more precise: exchanges
and queues that exist in remote clusters).

A federated exchange can route messages published upstream to a local queue. A federated
queue lets a local consumer receive messages from an upstream queue when the remote queue
itself does not have any consumers online.

Federation links connect to upstreams largely the same way an application would. Therefore
they can connect to a specific vhost, use TLS, use multiple
[authentication mechanisms](./authentication).

Federation documentation is organized as a number of more focussed guides:

 * [Exchange federation](./federated-exchanges): for replicating a flow of messages through an exchange to a remote cluster
 * [Queue federation](./federated-queues): to create a "logical queue" across N clusters that will move messages where consumers are (if there are no local consumers)
 * [Federation settings reference](./federation-reference)

## How is Federation Set Up? {#how-is-it-configured}

Two steps are involved in setting up federation:

* First, one or more upstreams must be defined. They provide federation with information about how to connect
  to other nodes. This can be done via [runtime parameters](./parameters)
  or the [federation management plugin](https://github.com/rabbitmq/rabbitmq-federation-management) which
  adds a federation management tab to the [management UI](./management).
* To enable federation, one or more [policies](./parameters#policies) that match exchanges or queues must be declared.
  The policy will make them federated.


## Getting Started {#getting-started}

The federation plugin is included in the RabbitMQ distribution. To
enable it, use [rabbitmq-plugins](./cli):

```bash
rabbitmq-plugins enable rabbitmq_federation
```

If [management UI](./management/) is used, it is recommended that
`rabbitmq_federation_management` is also enabled:

```bash
rabbitmq-plugins enable rabbitmq_federation_management
```

When using a federation in a cluster, all the nodes of the
cluster should have the federation plugin enabled.

Information about federation upstreams is stored in the RabbitMQ
database, along with users, permissions, queues, etc. There
are three levels of configuration involved in federation:

* **Upstreams**: each [upstream](./federation-reference#upstreams) defines a remote connection endpoint.
* **Upstream sets**: each [upstream set groups](./federation-reference#upstream-sets) together a set of upstreams to use for federation.
* **Policies**: each [policy](./parameters#policies) selects a set of exchanges,
  queues or both, and applies a single upstream or an upstream
  set to those objects.

In practice, for simple use cases you can almost ignore the
existence of upstream sets, since there is an implicitly-defined upstream set called `all`
to which all upstreams are added.

Upstreams and upstream sets are both instances of [runtime parameters](./parameters).
Like exchanges and queues, each virtual host has its own distinct set of parameters and policies. For more
generic information on parameters and policies, see the guide on
[parameters and policies](./parameters).
For full details on the parameters used by federation, see the [federation reference](./federation-reference).

Parameters and policies can be set in three ways:

 * Using [CLI tools](./cli)
 * In the management UI if an extension plugin (`rabbitmq_federation_management`) is enabled
 * Using the HTTP API

The HTTP API has a limitation: it does not support management of upstream sets.


## A Basic Example {#tutorial}

Here we will federate all the built-in exchanges except for
the default exchange, with a single upstream. The upstream
will be defined to buffer messages when disconnected for up
to one hour (3600000ms).

To define an upstream, use one of the following examples,
one per tab:

<Tabs>
<TabItem value="bash" label="bash" default>
```bash
rabbitmqctl set_parameter federation-upstream my-upstream \
    '{"uri":"amqp://target.hostname","expires":3600000}'
```
</TabItem>

<TabItem value="PowerShell" label="PowerShell">
```PowerShell
rabbitmqctl.bat set_parameter federation-upstream my-upstream `
    '"{""uri"":""amqp://target.hostname"",""expires"":3600000}"'
```
</TabItem>

<TabItem value="Management UI" label="Management UI">
Navigate to `Admin` > `Federation Upstreams` >
`Add a new upstream`. Enter "my-upstream" next to Name,
"amqp://target.hostname" next to URI, and 36000000 next to
Expiry. Click Add upstream.
</TabItem>

<TabItem value="HTTP API" label="HTTP API">
```bash
PUT /api/parameters/federation-upstream/%2f/my-upstream
{"value":{"uri":"amqp://target.hostname","expires":3600000}}
```
</TabItem>
</Tabs>

Then define a policy that will match built-in exchanges and use this upstream:

<Tabs>
<TabItem value="bash" label="bash" default>
```bash
rabbitmqctl set_policy --apply-to exchanges federate-me "^amq\." \
    '{"federation-upstream-set":"all"}'
```
</TabItem>

<TabItem value="PowerShell" label="PowerShell">
```PowerShell
rabbitmqctl.bat set_policy --apply-to exchanges federate-me "^amq\." `
    '"{""federation-upstream-set"":""all""}"'
```
</TabItem>

<TabItem value="Management UI" label="Management UI">
Navigate to `Admin` > `Policies` > `Add / update a policy`.
Enter "federate-me" next to "Name", "^amq\." next to
"Pattern", choose "Exchanges" from the "Apply to" drop down list
and enter "federation-upstream-set" = "all"
in the first line next to "Policy". Click "Add" policy.
</TabItem>

<TabItem value="HTTP API" label="HTTP API">
```ini
PUT /api/policies/%2f/federate-me
{"pattern":"^amq\.", "definition":{"federation-upstream-set":"all"}, "apply-to":"exchanges"}
```
</TabItem>
</Tabs>

The defined policy will make the exchanges _whose names
begin with "amq." (all the built-in exchanges except
for the default one) with (implicit) low priority, and
to federate them using the implicitly created upstream set
"all", which includes our newly-created upstream.

Any other [matching policy](./parameters#policies) with a priority greater than 0 will take
precedence over this policy. Keep in mind that `federate-me`
is just a name we used for this example, you can use any
string you want there.

The built in exchanges should now be federated because they are
matched by the policy. You can
check that the policy has applied to the exchanges by
checking the exchanges list in management or with:

```bash
rabbitmqctl list_exchanges name policy | grep federate-me
```

And you can check that federation links for each exchange have come up with `Admin` > `Federation Status` > `Running Links` or with:

```bash
# This command will be available only if federation plugin is enabled
rabbitmqctl federation_status
```

In general there will be one federation link for each
upstream that is applied to an exchange. So for example with
three exchanges and two upstreams for each there will be six
links.

For simple use this should be all you need - you will probably
want to look at the <a href="./uri-spec">AMQP URI
reference</a>.

The <a href="./federation-reference">federation reference</a> contains
more details on upstream parameters and upstream sets.


## Federation Connection (Link) Failures {#link-failures}

Inter-node connections used by Federation are based on AMQP 0-9-1
connections. Federation links can be treated as special kind of clients
by operators.

Should a link fail, e.g. due to a network interruption, it will
attempt to re-connect. Reconnection period is a configurable value
that's defined in upstream definition. See
<a href="./federation-reference">federation
reference</a> for more details on setting up upstreams and
upstream sets.

Links generally try to recover ad infinitum but there are scenarios
when they give up:

 * Failure rate is too high (max tolerated rate depends on
   upstream's `reconnect-delay` but is generally a failure
   every few seconds by default).
 * Link no longer can locate its "source" queue or exchange.
 * Policy changes in such a way that a link considers itself no longer necessary.

By increasing `reconnect-delay` for upstreams it is possible
to tolerate higher link failure rates. This is primarily relevant
for RabbitMQ installations where a moderate or large number of active links.


## Federating Clusters {#clustering}

Clusters can be linked together with federation just as single brokers
can. To summarise how clustering and federation interact:

 * You can define policies and parameters on any node in the downstream
   cluster; once defined on one node they will apply on all nodes.
 * Exchange federation links will start on any node in the
   downstream cluster. They will fail over to other nodes if
   the node they are running on crashes or stops.
 * Queue federation links will start on the same node as the
   downstream queue. If the downstream queue is a [replicated one](./quorum-queues), they
   will start on the same node as the leader, and will be
   recreated on the same node as the new leader after any future leader elections.
 * To connect to an upstream cluster, you can specify multiple URIs in
   a single upstream. The federation link process will choose one of
   these URIs at random each time it attempts to connect.


## Securing Federation Connections with TLS {#tls-connections}

Federation connections (links) can be secured with TLS. Because Federation uses
a RabbitMQ client under the hood, it is necessary to both configure
source broker to [listen for TLS connections](./ssl)
and Federation/Erlang client to use TLS.

To configure Federation to use TLS, one needs to

 * Use the `amqps` URI scheme instead of `amqp`
 * Specify CA certificate and client certificate/key pair via [URI query parameters](./uri-query-parameters)
   when configuring upstream(s)
 * [Configure Erlang client to use TLS](./ssl)

Just like with "regular" client connections, server's CA should be
trusted on the node where federation link(s) runs, and vice versa.


## Federation Link Monitoring {#status}

Each combination of federated exchange or queue and upstream needs a
link to run. This is the process that retrieves messages from upstream
and republishes them downstream. You can monitor the status of
federation links using `rabbitmqctl` and the management
plugin.

### Using CLI Tools

Federation link status can be inspected using [RabbitMQ CLI tools](./cli).

Invoke:

```bash
# This command will be available only if federation plugin is enabled
rabbitmqctl federation_status
```

This will output a list of federation links running on the target node (not cluster-wide).
It contains the following keys:

<table>
  <thead>
    <tr>
      <td>Parameter Name</td>
      <td>Description</td>
    </tr>
  </thead>

  <tbody>
    <tr>
      <td>`type`</td>
      <td>
        `exchange` or `queue` depending on
        what type of federated resource this link relates to
      </td>
    </tr>

    <tr>
      <td>`name`</td>
      <td>
        the name of the federated exchange or queue
      </td>
    </tr>

    <tr>
      <td>`vhost`</td>
      <td>
        the virtual host containing the federated exchange or queue
      </td>
    </tr>

    <tr>
      <td>`upstream_name`</td>
      <td>
        the name of the upstream this link is connected to
      </td>
    </tr>

    <tr>
      <td>`status`</td>
      <td>
        status of the link:
          <ul>
            <li>`starting`</li>
            <li>`{running, LocalConnectionName}`</li>
            <li>`{shutdown, Error}`</li>
          </ul>
      </td>
    </tr>

    <tr>
      <td>`connection`</td>
      <td>
        the name of the connection for this link (from config)
      </td>
    </tr>

    <tr>
      <td>`timestamp`</td>
      <td>
        time stamp of the last status update
      </td>
    </tr>
  </tbody>
</table>

Here's an example:

```bash
# This command will be available only if federation plugin is enabled
rabbitmqctl federation_status
# => [[{type,<<"exchange">>},
# =>   {name,<<"my-exchange">>},
# =>   {vhost,<<"/">>},
# =>   {connection,<<"upstream-server">>},
# =>   {upstream_name,<<"my-upstream-x">>},
# =>   {status,{running,<<"<rabbit@my-server.1.281.0>">>}},
# =>   {timestamp,{{2020,3,1},{12,3,28}}}]]
# => ...done.
```

### Using the Management UI

Enable the `rabbitmq_federation_management` [plugin](./plugins) that extends
[management UI](./management) with a new page that displays federation links in the cluster.
It can be found under `Admin` > `Federation Status`, or by using the
`GET /api/federation-links` HTTP API endpoint.
