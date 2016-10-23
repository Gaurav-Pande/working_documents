## Getting a complete configuration for MDT with Native YANG and YDK

In an [earlier tutorial](https://xrdocs.github.io/telemetry/tutorials/2016-08-08-configuring-model-driven-telemetry-with-ydk/), Shelly introduces a methodology to configure MDT using YDK and the OpenConfig Telemetry YANG model.

I strongly suggest to read and understand her initial tutorial because it describes the basis of YDK. 
I decided to prepare this second document, covering a similar subject only because the OpenConfig Telemetry YANG model is still incomplete and you will not be able to set the protocol and encoding fields in a destination group. 

OpenConfig YANG models are the preferred option for POCs, demostrating an dopen strategy. At the same time, if you need to demonstrate or implement a complete (working) MDT dial-out configuration using YANG models, you must use a IOS XR Native YANG model using YDK (described in this tutorial) or XML schema as Shelly describes in [Configuring MDT with OpenConfig YANG] (https://xrdocs.github.io/telemetry/tutorials/2016-07-25-configuring-model-driven-telemetry-mdt-with-yang/). The engineering team is working to finalise a complete OpenConfig YANG model for our XR telemetry configuration but this may take some XR releases.

Note: I have tested the configuration proposed in this document using IOS-XRv version 6.2.1.15I, noticing an issue with earlier versions that accept but doesn't implement the destination group TCP protocol configuration.

If you are not familiar with IOS-XRv, please follow Akshat tutorial [IOS-XRv Vagrant Quick Start](https://xrdocs.github.io/application-hosting/tutorials/iosxr-vagrant-quickstart) for step by step instructions.

## Tutorial goal

By the end of this tutorial, you will have implemented the follwoing configuration using YDK on your router under testing. If you are unfamiliar with the configration just check the following tutorial [Configuring Model-Driven Telemetry (MDT)](https://xrdocs.github.io/telemetry/tutorials/2016-07-21-configuring-model-driven-telemetry-mdt/)

{% capture "output" %}
CLI Output:

```
P/0/RP0/CPU0:test_XR#show running-config telemetry model-driven 
Fri Oct 21 06:51:06.926 UTC
telemetry model-driven
 destination-group DG_Test
  address family ipv4 192.168.10.3 port 5432
   encoding self-describing-gpb
   protocol tcp
  !
 !
 sensor-group SG_Test
  sensor-path Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters/bytes-received
 !
 subscription 1
  sensor-group-id SG_Test sample-interval 30000
  destination-id DG_Test
 !
!
``` 

{% endcapture %}


## Connect to the router and import YDK's library

As described in older YDK tutorails, we are importing the YDK Netconf library to cominicate with the router and other key YDK's library like the CRUDService (taking care of create, read, update and delete YDK objects from the router) and the IOS_XR native YDK model.
The Empty type that we import from ydk.types has a special purpose later in the document to signal with its presence, the request to activate the submitted subcription.

```python
from ydk.providers import NetconfServiceProvider 
from ydk.services import CRUDService
from ydk.types import Empty 
import ydk.models.cisco_ios_xr.Cisco_IOS_XR_telemetry_model_driven_cfg as xr_telemetry

HOST = '192.168.10.2'
PORT = 830
USER = 'vagrant'
PASS = 'vagrant'

xr = NetconfServiceProvider(address=HOST,
	port=PORT,
	username=USER,
	password=PASS,
	protocol = 'ssh')
```

With that, we are now connected to the router (check on the router):

{% capture "output" %}
CLI Output:

```
RP/0/RP0/CPU0:test_XR#show netconf-yang clients 
Fri Oct 21 01:36:02.850 UTC
Netconf clients
client session ID|     NC version|    client connect time|        last OP time|        last OP type|    <lock>|
       4261169968|            1.1|         0d  0h  0m 28s|                    |                    |        No|
RP/0/RP0/CPU0:test_XR#

``` 

{% endcapture %}

<div class="notice--info">
{{ output | markdownify }}
</div>


## Define and apply the destination group



{% capture "output" %}
PYANG Output:

```
module: openconfig-telemetry
   +--rw telemetry-system
      +--rw sensor-groups
      |  +--rw sensor-group* [sensor-group-id]
      |     +--rw sensor-group-id    -> ../config/sensor-group-id
      |     +--rw config
      |     |  +--rw sensor-group-id?   string
   
```  
{% endcapture %}

<div class="notice--warning">
{{ output | markdownify }}
</div>


```python
dgroup=xr_telemetry.TelemetryModelDriven.DestinationGroups.DestinationGroup()
dgroup.destination_id="DG_Test"
dgroup.destinations=dgroup.Destinations()

new_destination=dgroup.Destinations.Destination()
new_destination.address_family=xr_telemetry.AfEnum.IPV4

new_ipv4=xr_telemetry.TelemetryModelDriven.DestinationGroups.DestinationGroup().Destinations().Destination().Ipv4()
new_ipv4.destination_port=5432
new_ipv4.ipv4_address="192.168.10.3"
new_ipv4.encoding=xr_telemetry.EncodeTypeEnum.SELF_DESCRIBING_GPB
new_ipv4.protocol=xr_telemetry.TelemetryModelDriven.DestinationGroups.DestinationGroup().Destinations().Destination().Ipv4().Protocol()
new_ipv4.protocol.protocol=xr_telemetry.ProtoTypeEnum.TCP
new_destination.ipv4.append(new_ipv4)
dgroup.destinations.destination.append(new_destination)

```
Once you’ve populated the object, we can apply it to the router using the create method on the CRUDService object from YDK:

```python
rpc_service = CRUDService()
rpc_service.create(xr, dgroup)
```

And here is the expected CLI output with the destination group describing where and how to steam:

{% capture "output" %}
CLI Output:

```
RP/0/RP0/CPU0:test_XR#sh running-config telemetry model-driven 
Fri Oct 21 06:35:06.731 UTC
telemetry model-driven
 destination-group DG_Test
  address family ipv4 192.168.10.3 port 5432
   encoding self-describing-gpb
   protocol tcp
  !
 !
!
``` 

{% endcapture %}

<div class="notice--info">
{{ output | markdownify }}
</div>


## Define and apply the sensor group

```python
sgroup = xr_telemetry.TelemetryModelDriven.SensorGroups.SensorGroup()
sgroup.sensor_group_identifier="SG_Test"

sgroup.sensor_paths = sgroup.SensorPaths()
new_sensorpath = sgroup.SensorPaths.SensorPath()
new_sensorpath.telemetry_sensor_path = 'Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters/bytes-received'
sgroup.sensor_paths.sensor_path.append(new_sensorpath)
```

Now we are ready to submit the sensor group object just populated with the same create method from the CRUDService object of YDK:

```python

rpc_service.create(xr, sgroup)

```

Note: you need to initilialise the rpc_service as `rpc_service = CRUDService()` a single time. If you remember, we have done it when creating the destinatin group at the previous step but if you skipped the previous step, add it before requeting the create for the sensor group.

Let's check the CLI running-configuraiton again. You shoudl now find the destination (from the previous step) and sensor group just created in place:

{% capture "output" %}
CLI Output:

```
RP/0/RP0/CPU0:test_XR#show running-config telemetry model-driven 
Fri Oct 21 06:45:58.444 UTC
telemetry model-driven
 destination-group DG_Test
  address family ipv4 192.168.10.3 port 5432
   encoding self-describing-gpb
   protocol tcp
  !
 !
 sensor-group SG_Test
  sensor-path Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters/bytes-received
 !
!
``` 

{% endcapture %}

<div class="notice--info">
{{ output | markdownify }}
</div>


## Define and apply the subcription

```python
sub = xr_telemetry.TelemetryModelDriven.Subscriptions.Subscription()
sub.subscription_identifier = "1"

sub.sensor_profiles = sub.SensorProfiles()
sub.destination_profiles = sub.DestinationProfiles()
                               
new_sprofile = sub.SensorProfiles.SensorProfile()
new_sprofile.sensorgroupid = 'SG_Test'
new_sprofile.sample_interval = 30000

new_dprofile = sub.DestinationProfiles.DestinationProfile()
new_dprofile.destination_id="DG_Test"
new_dprofile.enable=Empty()

sub.sensor_profiles.sensor_profile.append(new_sprofile)
sub.destination_profiles.destination_profile.append(new_dprofile)
```

```python
rpc_service.create(xr, sub)
```

If you check now the running-configuration, you should have a complete TCP dial-out telemetry configuation

{% capture "output" %}
CLI Output:

```
P/0/RP0/CPU0:test_XR#show running-config telemetry model-driven 
Fri Oct 21 06:51:06.926 UTC
telemetry model-driven
 destination-group DG_Test
  address family ipv4 192.168.10.3 port 5432
   encoding self-describing-gpb
   protocol tcp
  !
 !
 sensor-group SG_Test
  sensor-path Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters/bytes-received
 !
 subscription 1
  sensor-group-id SG_Test sample-interval 30000
  destination-id DG_Test
 !
!
``` 

{% endcapture %}

<div class="notice--info">
{{ output | markdownify }}
</div>

## Clean

Last two commands proposed support deleting the configuration just submitted and disconnect from the router the Netconf session.

```python
rpc_service.delete(xr, xr_telemetry.TelemetryModelDriven())
xr.close()
```
As condirmed by the router's 'show running-config'

{% capture "output" %}
CLI Output:

```
RP/0/RP0/CPU0:test_XR#show running-config telemetry model-driven 
Fri Oct 21 06:59:46.743 UTC
% No such configuration item(s)

RP/0/RP0/CPU0:test_XR#

``` 

{% endcapture %}

<div class="notice--info">
{{ output | markdownify }}
</div>

## Conclusion

This tutorial repeats most of the concepts explained by Shelly in her [earlier tutorial](https://xrdocs.github.io/telemetry/tutorials/2016-08-08-configuring-model-driven-telemetry-with-ydk/). The same values in programmability using YANG models, automatic generation of Python classes that inherit the syntactic checks and requirements of the underlying model, while also handling all the details of the underlying encoding and transport.

At the same time it provides an alternative YANG model that at the time of writing is the only option to demostrate a working XR telemetry dial-out solution using YDK. 
I hope this will be usefull when preparing POC or just learning YDK and telemetry.

