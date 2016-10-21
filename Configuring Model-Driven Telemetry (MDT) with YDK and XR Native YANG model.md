## Getting a complete configuration for MDT with Native YANG and YDK

In an [earlier tutorial](https://xrdocs.github.io/telemetry/tutorials/2016-08-08-configuring-model-driven-telemetry-with-ydk/), Shelly introduces a methodology to configure MDT using YDK and the OpenConfig Telemetry YANG model.

I strongly suggest to read and understand her initial tutorial because it describes the basis of YDK. 
I decided to prepare this second document, covering a similar subject only because the OpenConfig Telemetry YANG model is still incomplete and you will not be able to set the protocol and encoding fields in a destination group. 

OpenConfig YANG models are the preferred option for POCs (showing openess in our strategy). At the same time, if you need to demonstrate a working MDT dial-out configuration using YANG models, you must use a IOS XR Native YANG model using YDK (described in this tutorial) or XML schema as Shelly describes in [Configuring MDT with OpenConfig YANG] (https://xrdocs.github.io/telemetry/tutorials/2016-07-25-configuring-model-driven-telemetry-mdt-with-yang/). The engineering team is working to get published a complete OpenConfig for our XR telemetry configuration but this may take a while.

Note: I have tested the configuration proposed in this document using IOS-XR version 6.2.1.15I, noticing an issue with earlier versions discarding the destination group TCP protocol configuration.

If you are not familiar with IOS-XR, please follow Akshat tutorial [IOS-XR Vagrant Quick Start](https://xrdocs.github.io/application-hosting/tutorials/iosxr-vagrant-quickstart) for step by step instructions.

## Connect to the router

```python
from ydk.providers import NetconfServiceProvider
from ydk.services import CRUDService
from ydk.types import Empty 
import ydk.models.cisco_ios_xr.Cisco_IOS_XR_telemetry_model_driven_cfg as xr_telemetry

HOST = '192.168.1.2'
PORT = 830
USER = 'vagrant'
PASS = 'vagrant'

xr = NetconfServiceProvider(address=HOST,
	port=PORT,
	username=USER,
	password=PASS,
	protocol = 'ssh')
```

With that, we are now connected to the router:

{% capture "output" %}
CLI Output:

```
RP/0/RP0/CPU0:SunC#show netconf-yang clients
Mon Aug  8 23:01:48.210 UTC
Netconf clients
client session ID|     NC version|    client connect time|        last OP time|        last OP type|    <lock>|
       1386485520|            1.1|         0d  0h  0m  5s|                    |                    |        No|
RP/0/RP0/CPU0:SunC#
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
new_ipv4.ipv4_address="192.168.1.1"
new_ipv4.encoding=xr_telemetry.EncodeTypeEnum.SELF_DESCRIBING_GPB
new_ipv4.protocol=xr_telemetry.TelemetryModelDriven.DestinationGroups.DestinationGroup().Destinations().Destination().Ipv4().Protocol()
new_ipv4.protocol.protocol=xr_telemetry.ProtoTypeEnum.TCP
new_destination.ipv4.append(new_ipv4)
dgroup.destinations.destination.append(new_destination)

```

```python
rpc_service = CRUDService()
rpc_service.create(xr, dgroup)
```

## Define and apply the sensor group

```python
sgroup = xr_telemetry.TelemetryModelDriven.SensorGroups.SensorGroup()
sgroup.sensor_group_identifier="SG_Test"

sgroup.sensor_paths = sgroup.SensorPaths()
new_sensorpath = sgroup.SensorPaths.SensorPath()
new_sensorpath.telemetry_sensor_path = 'Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters/bytes-received'
sgroup.sensor_paths.sensor_path.append(new_sensorpath)
```
```python
rpc_service = CRUDService()
rpc_service.create(xr, sgroup)
```
Note: you need to initilialize the rpc_service a single time, if you read this document sequentially we have done when creating the destination group and you can remove it


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
rpc_service = CRUDService()
rpc_service.create(xr, sub)
```


## Clean

```python
rpc_service = CRUDService()
rpc_service.delete(xr, xr_telemetry.TelemetryModelDriven())
xr.close()
```

## Conclusion

