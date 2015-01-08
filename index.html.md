---
title: Redis for Pivotal CF
---

This is documentation for the Redis service for Pivotal CF.

## Install via Pivotal Operations Manager

To install Redis for Pivotal CF, follow the procedure for installing Pivotal Ops Manager tiles:

1. Download the product file from [Pivotal Network](https://network.pivotal.io/).
1. Upload the product file to your Ops Manager installation.
1. Click **Add** next to the uploaded product description in the Available Products view to add this product to your staging area.
1. Click the newly added tile to review any configurable options.
1. Click **Apply Changes** to install the service.

## Dependencies
This product requires Pivotal CF:

* Elastic Runtime version 1.2 or greater
* Ops Manager version 1.3 or greater


## Available Plans

There are two available plans:

<table border="1" class="nice">
<tr>
<th><strong>Plan Name</strong></th>
<th><strong>Suitable for</strong></th>
<th><strong>Tenancy Model per Instance</strong></th>
<th><strong>Highly Available</strong></th>
<th><strong>Backup Functionality</strong></th>
</tr>

<tr>
<td><b>Shared-VM</b></td>
<td>Development and testing workloads</td>
<td>Shared VM</td>
<td>No</td>
<td>No</td>
</tr>

<tr>
<td><b>Dedicated-VM</b></td>
<td>Production workloads</td>
<td>Dedicated VM</td>
<td>No</td>
<td>No</td>
</tr>

</table>

## Provisioning and Binding via Cloud Foundry

Once you have installed the product, it automatically registers itself with your Elastic Runtime. At this point, the product is available to your application developers, either in the Marketplace in the web based console, or via `cf marketplace`. They can add, provision, and bind the service to their applications like any other CF service:

```
$ cf create-service p-redis shared-vm redis
$ cf bind-service my-application redis
$ cf restart my-application
```

## Example Application

To help your application developers get started with Redis for Pivotal CF, we have provided an example application, which can be [downloaded here](https://github.com/pivotal-cf/cf-redis-example-app/archive/master.zip).

## Feedback

Please provide any bugs, feature requests, or questions to [the Pivotal CF Feedback list](mailto:pivotal-cf-feedback@pivotal.io).

## Version

This product is based on Redis version 2.8.19. 

## Further Reading

* [Official Redis Documentation](http://redis.io/documentation)

