---
title: Redis for PCF 1.9 Architecture and Lifecycle
owner: London Services
---

## <a id ="config"></a>How Redis for PCF Configures Redis for On-Demand Service Plans

For On-Demand Plans, certain Redis configurations can be set by the operator during plan configuration,
 and by the App Developer during instance provisioning. Other Redis configurations cannot be changed from the default.

 + Operator configurable Redis settings include: `timeout`, `tcp-keepalive`, `maxclients` and `lua scripting`. Please see the Operator Guide section of this documentation for more detail.
 + App-Developer configurable Redis settings include: `maxmemory-policy`, `notify-keyspace-events`,`slowlog-log-slower-than`,and `slowlog-max-len`. Please see the App Developer guide of this documentation for more detail.

## How Redis for PCF Configures Redis for Dedicated-VM and Shared-VM Service Plans

For Dedicated-VM and Shared-VM plans, the default Redis configurations cannot be changed.
A sample `redis.conf` from a Dedicated-VM plan instance can be viewed on [Sample Redis Configuration] (redisconf.html)

+ Redis is configured with a `maxmemory-policy` of `no-eviction`. This policy means that when the memory is full, the service does
 not evict any keys or perform any write operations until memory becomes available.

+ Persistence is configured for both `RDB` and `AOF`.

+ The default maximum number of connections, `maxclients`, is set at 10000 but
this number is adjusted by Redis according to the number of file handles available.

+ Replication and event notification are not configured.


##  <a id="plans"></a>Service Plans

Redis for PCF offers On-Demand, Dedicated-VM and Shared-VM plans.

### On-Demand Service Plans
* These plans are Operator-configured and enabled. Once enabled, App Developers can provision a Redis instance from that plan.
* You can disable any of the three on-demand plans in the plan's page in OpsManager.
* The maximum number of instances is managed at by a per-plan and global quota. The maximum number of instances cannot surpass 50.
* Operators can update the plan settings, including the VM size, disk size and Redis configuration settings, after the plans have been created. **Operators should not downsize the VMs or disk size as this can cause data loss in pre-existing instances.**
* App Developers can update certain Redis configurations.

### Shared-VM Service Plan

* This plan deploys a Redis instance inside the service broker VM.
* You can disable this plan by setting the `Max instances limit` on the `Shared-VM plan` tab in OpsManager to be `0`.
* The maximum number of instances can be increased from the default 5 to a value of your choosing. If you increase the number of instances that can be run on this single VM, you should consider increasing the resources allocated to the VM. In particular RAM and CPU. You can overcommit to a certain extent, but may start to see performance degradations.
* You can also increase the maximum amount of RAM allocated to each Redis process (service instance) that is running on this VM
* If you decrease the service instance limit, any instances that are running where the count is now greater than the limit are not terminated. They are left to be removed naturally, until the total count drops below the new limit you cannot create any new instances. For example if you had a limit of 10 and all were used and reduced this to 8, the two instances will be left running until you terminate them yourself.

### Dedicated-VM Service Plan

* This plan deploys the operator-configured number of dedicated Redis VMs alongside the service broker VM.
* These instances are pre-provisioned during the deployment of the tile from OpsManager into a **pool**. The VMs are provisioned and configured with a Redis process ready to be used when an instance of the Dedicated-VM plan is requested.
* A default deployment will provision `5 instances` of the Dedicated-VM plan into the **pool**. This number can be increased on the `Resource Config` tab in Ops Manager, either in the initial deployment, or subsequently thereafter. The number of VMs **cannot** be decreased once deployed.
* When a user provisions an instance, it is marked as in use and taken out of the **pool**.
* When a user deprovisions an instance, the instance is cleansed of any data and configuration to restore it to a fresh state and placed back into the pool, ready to be used again.
* You can disable this plan by setting the number of instances of the `Dedicated node` job in Ops Manager to `0`.

##  <a id="downtime"></a> Downtime for Redis

Downtime is caused by a redeploy of the Redis tile. Here we attempt to clarify what changes will trigger a redeploy.

In Ops Manager, any field that changes the manifest will cause a redeploy of the Redis tile. Any property changes in ERT listed here will trigger downtime for the Redis On-Demand Broker. Note that this list is current at the time of writing (May 2017) but that dependencies with changing products mean that it will change.
<ul>
  <li>Consul Server Resource Config (..cf.consul\_server.ips)</li>
  <li>Runtime System Domain ($runtime.system\_domain)</li>
  <li>Disable SSL certificate verification for this environment” in ERT
(..cf.ha\_proxy.skip\_cert\_verify.value)</li>
  <li>Runtime Apps Domain ($runtime.apps\_domain)</li>
  <li>NATS Resource Config
(..cf.nats.ips)</li>
  <li>Service Networks in OpsManager
($self.service\_network)</li>
</ul>


For Redis for PCF v1.8+, downtime for service instances will only occur once the Operator runs `upgrade-all-service-instances` BOSH errand after all tile upgrades have completed successfully. For Redis for PCF v1.7 and earlier, and the Dedicated-VM and Shared-VM Service in Redis for PCF v1.8 and v1.9, downtime is enforced as soon as the operator applies the referenced changes to ERT. Any changes to a field on the Redis tile, will cause BOSH to redeploy the legacy Redis Broker and the On-Demand Broker once `upgrade-all-service-instances` is run.


## <a id ="architecture"></a>Redis for PCF Architecture for On-Demand Service Plan

This diagram shows how the architecture of the service broker and On Demand plans and how the user's app binds to a Redis instance.

![On-Demand Architecture Diagram](on-demand-arch.png)

### <a id ="architecture_networks"></a>Default Network and Service Network

Like other on-demand PCF services, on-demand Redis for PCF relies on the BOSH 2.0 ability to dynamically deploy VMs in a dedicated network. The on-demand service broker uses this capability to create single-tenant service instances in a dedicated service network.

On-demand services use the dynamically-provisioned service network to host the single-tenant worker VMs that run as service instances within development spaces. This architecture lets developers provision IaaS resources for their service instances at creation time, rather than the operator pre-provisioning a fixed quantity of IaaS resources when they deploy the service broker.

By making services single-tenant, where each instance runs on a dedicated VM rather than sharing VMs with unrelated processes, on-demand services eliminate the “noisy neighbor” problem when one application hogs resources on a shared cluster. Single-tenant services can also support regulatory compliance where sensitive data must be compartmentalized across separate machines. Redis for PCF offers another single-tenant plan, the Dedicated-VM plan, which does not offer the same beneifts

An on-demand service splits its operations between the default network and the service network. Shared components of the service, such as executive controllers and databases, run centrally on the default network along with the Cloud Controller, UAA, and other PCF components. The worker pool deployed to specific spaces runs on the service network.

### BOSH 2.0 and the Service Network

Before BOSH 2.0, cloud operators pre-provisioned service instances from Ops Manager. This is used in the Redis Dedicated-VM and Shared-VM plan. In the Ops Manager Director Networking pane, they allocated a block of IP addresses for the service instance pool, and under Resource Config they provisioned pool VM resources, specifying the CPU, hard disk, and RAM they would use. All instances had to be provisioned at the same level. With each `create-service` request from a developer, Ops Manager handed out a static IP address from this block, and with each `delete-service` it cleaned up the VM and returned it to the available pool.

With BOSH 2.0 dynamic networking and Cloud Foundry asynchronous service provisioning, operators can now define a dynamically-provisioned service network that hosts instances more flexibly. The service network runs separate from the PCF default network. While the default network hosts VMs launched by Ops Manager, the VMs running in the service network are created and provisioned on-demand by BOSH, and BOSH lets the IaaS assign IP addresses to the service instance VMs. Each dynamic network attached to a job instance is typically represented as its own NIC (Network Interface Controller) in the IaaS layer.

Operators enable on-demand services when they deploy PCF, by creating one or more service networks in the Ops Manager Director Create Networks pane and selecting the Service Network checkbox. Designating a network as a service network prevents Ops Manager from creating VMs in the network, leaving instance creation to the underlying BOSH.

![Architecture Diagram](ODB-architecture.png)

## Redis for PCF Architecture for Dedicated-VM and Shared-VM Service Plans

This diagram shows how the architecture of the service broker and Shared-VM and Dedicated-VM plans and how the user's app binds to a Redis instance.

![Architecture Diagram](legacy-arch.png)

## <a id ="lifecycle"></a>Redis for PCF Lifecycle for On-Demand Service Plan

Here is the lifecycle of Redis for PCF, from an operator installing the tile through an app developer using the service then an operator deleting the tile.

![On-Demand Lifecycle Diagram](Redis_timeline_demand.png)

## Redis for PCF Lifecycle for Dedicated-VM and Shared-VM Service Plans

Here is the lifecycle of Redis for PCF, from an operator installing the tile through an app developer using the service then an operator deleting the tile.

![Lifecycle Diagram](Redis_timeline_legacy.png)
