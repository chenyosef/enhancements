---
title: Baremetal IPI Config Enhancement
authors:
  - "@sadasu"
reviewers:
  - "@deads2k"
  - "@hardys"
  - "@enxebre"
  - "@squeed"
  - "@abhinavdahiya"
  - "@dhellmann"

approvers:
  - "@abhinavdahiya"
  - "@enxebre"
  - "@deads2k"

creation-date: 2019-11-14
last-updated: 2019-11-26
status: not implemented
see-also:
  - 
replaces:
  - https://github.com/openshift/enhancements/pull/90
superseded-by:
  - 
---

# Config required for Baremetal IPI deployments

## Release Signoff Checklist

- [*] Enhancement is `implementable`
- [*] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [openshift/docs]

## Open Questions

1. Which is a preferred name for the new CR "Metal3ProvisioningController" or
"Metal3Controller"?

[Closed] Based on review comments it appears that "Metal3ProvisioningController"
is the preferred name since it makes the contents of the config very clear.

2. There is a possibility that the MCO would also have to refer to this new CR
created in operator scope. Would it be OK for the MCO to access the CR with this
(limited) scope?

[Closed] MCO will be able to read config from this new CR if needed.

## Summary

The configuration representing different provisioning network parameters used
for provisioning baremetal servers need to be made available to the Machine
API Operator (MAO) in a Config Resource. We had submitted an earlier proposal
[1] to augment the BareMetalInfrastructureStatus CR with these config items.
Based on feedback we recieved we are revising our proposal to create a new CR
for these configuration items.

This new CR needs to be accessed by just the Installer and the Machine API
Operator and hence does not need to be created with global scope. The installer
would be responsible for instantiating the CR with user input values. The MAO is
responsible for deploying the metal3 cluster. And the configuration values from
this CR would be read by the MAO and used to generate some other configuration
parameters. All of these values (user provided and MAO derived) need to be
passed in as env vars to the containers that are part of the metal3 cluster
before starting the containers. In most cases, the containers will not start
without these init configs available as env vars. 

For a background on the work being done for BareMetal IPI installs please refer
to [2] for the necessary context for the enhancements proposed here.

## Motivation

The Baremetal IPI deployments are different from the other platform types currently
being supported by OpenShift in that there is no underlying cloud platform
exposing an API as in public clouds e.g AWS, until the Baremetal Operator (BMO)
along with Ironic services are run exposing an inventory of available hosts as
custom resources.

The "metal3" pod deployed by the Machine API operator (MAO) contains the BareMetal
Operator (BMO) and a provisioning service (Ironic) that are together responsible for
PXE booting baremetal servers and enrolling them as BareMetal Hosts. The MAO is
responsible for deploying the BMO and the Ironic containers.

This enhancement request proposes to add a new CR for configs required only for
the provisioning service to PXE boot a baremetal server and make it available as a
BareMetal host. The configurations in this baremetal provisioning CR, are seperate
from the configuration in the BareMetalHost CR. This new CR will allow for setting
some sane defaults for these configurations with the ability to overrride them if 
required. After the baremetal server is provisioned, the configurations in the
BareMetalHost CR contain information to connect to the controller and also to
fulfill a Machine.

### Goals

The goal of this enhancement request is to provide details about the configuration
being added to a new CR used only in "metal3" deployments. This new CR would be
within the operator scope and will only contain configuration required by the
provisioning service to boot baremetal servers.

### Non-Goals

The provisioning network or the provisioning IP are not expected to change after
deployment. But, it is possible for the DHCP range to be expanded after nodes
have already been deployed. This proposal is not considering the update path for
these configurations items.

### Proposal

Before BareMetal Hosts can be matched to Machines, they need to be connected to their
provisioning network for them to be PXE booted and given an IP address. To make
this happen, the provisioning service needs to know which NIC, provisioning network,
IP address and image URLs to use to download and boot images on these servers.

A new CR called "Metal3ProvisioningController" would be created within the operator scope.

This new CR would consist of the following:

1. ProvisioningInterface : This is the interface name on the underlying baremetal
server which would be physically connected to the provisioning network. This
configuration is needed only for the underlying provisioning service (Ironic)
and could have values like "eth1" or "ens3".

2. ProvisioningIP : This is the IP address used to to bring up a NIC on the
baremetal server for provisioning. This is also a value that is useful just to the
provisioning service. This value should not be in the DHCP range and should not
be used in the provisioning network for any other purpose (should be a free IP
address.) It is expected to be provided in the format : 10.1.0.3.

3. ProvisioningNetworkCIDR : This is the provisioning network with its CIDR. The
ProvisioningIP and the ProvisioningDHCPRange are expected to be within this network.
The value for this config is expected to be provided in the format : 10.1.0.0/24.

4. ProvisioningDHCP : This configuration needs to convey two values: if the DHCP
service needs to be managed within the cluster and if so what is the range of IP
addresses that can used. Towards that end, this configuration would be a struct
with 2 members:
	1. ManagementType - The ManagementType is a string that can have any of
	the following states "Managed", "Unmanaged", "Force", "Removed". In this
	case, we are interested only in 2 states, "Managed" and "Removed". If the
	ManagementType is "Removed" it means that the metal3 cluster would not be
	responsible for managing DHCP addresses and an external DHCP server is
	expected to be available and reachable by the cluster. If the ManagementType
	is "Managed", then the DHCP range indicates the pool of IP addresses that
	can used to assign to the baremetal hosts. This value cannot be changed
	after installation.

	2. DHCPRange - The DHCPRange when set, is a string which consists of a pair of
	comma seperated IP addresses representing the start and end of the IP address
	range. If unset, then the default IP address range (.10 to .100) would be
	used. The value of the DHCP range can be changed even after insallation.

### User Stories [optional]

1. As a Deployment Operator, I want Barametal IPI deployments to be customizable to
hardware and network requirements.

2. As an OpenShift Administrator, I want Baremetal IPI deployments to take place without
manual workarounds like creating a ConfigMap for the config (which is the current approach
being used in 4.2 and 4.3.) 

## Design Details

This new baremetal CR would be created in the "openshift-machine-api" namespace and as
mentioned earlier would be in operator scope. Only one instance of this CR would be
created by the installer and hence it is a singleton CR.

Important details of the CR:

Resource name - provisioningconfig.baremetal.operator.openshift.io
Instance name - main/default
Namespace - openshift-machine-api
Version - apiextensions.k8s.io/v1

The new config items would be set by the installer and will be used by the MAO to
generate more config items that are derivable from these basic parameters. Put
together, these config parameters are passed in as environment variables to the various
containers that make up a metal3 baremetal IPI deployment.

This baremetal provisioning CR contains configuration data for the provisioning services,
which are not values that should be configured by the end user via BareMetalHost objects.

The configs described in this enhancement doc would be part of the Spec field of the CR.
Only the ProvisioningDHCP.DHCPRange field can change after installtion, so this will be
marked as editable. All other config items will be marked as not editable.
 
### Test Plan

The test plan should involve making sure the openshift/installer generates
all configuration items within the BaremetalPlatformStatus when the platform
type is Baremetal.

MAO reads this configuration and uses these to derive additional configuration
required to bring up a metal3 cluster. E2e testing should make sure that MAO
is able to bring up a metal3 cluster using config from this new Metal3Controller
CR which has operator scope.

Once metal3 is up, the next level of testing should involve bringing up worker nodes.
Also, testing needs to make sure we are still able to bring up worker nodes when there
is an external DHCP server and we donot bring up DHCP services within the cluster.

Test plan should also include tests to dynamically increase the DHCP range after a
metal3 cluster has been up and a few workers have come up successfully.

### Upgrade / Downgrade Strategy

Baremetal Platform type will be available for customers to use for the first
time in OpenShift 4.3. And, when it is installed, it will always start as a
fresh baremetal installation at least in 4.3. There is no use case where a 4.2
installation would be upgraded to a 4.3 installation with Baremetal Platform
support enabled.

To ensure a hitless upgrade from 4.3 to 4.4, the implementation in 4.4 would try to
read the configuration from the new CR and the ConfigMap. If MAO is unable to find the
provsioning configuration in the new CR, it will fallback to reading it from the ConfigMap.
And, this decision will be made per config item and not based just on the presence of the
new CR. 

[1] - https://github.com/openshift/enhancements/pull/90
[2] - https://github.com/openshift/enhancements/pull/102
