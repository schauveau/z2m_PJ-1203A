# z2m_PJ-1203A
A small repository for my custom zigbee2mqtt converter for the PJ-1203A

I started a discussion in https://github.com/Koenkk/zigbee2mqtt/discussions/21956

## Installation

Assuming that zigbee2mqtt is installed in `/opt/zigbee2mqtt`, copy one of the proposed variant 
to `/opt/zigbee2mqtt/data/PJ-1203A.js`. 
 
The external converter also need to be declared in `/opt/zigbee2mqtt/configuration.yaml` or in 
the web interface.
 
Be aware that `zigbee2mqtt` will fallback to the builtin `PJ-1203A` converter if the external 
file cannot be loaded or contains errors.

Syntax errors in the converters can be detected before starting zigbee2mqtt with 

    node /opt/zigbee2mqtt/data/PJ-1203A.js
    
but be aware that this trick only works for files inside the `/opt/zigbee2mqtt` directory.

Your systemd service file for zigbee2mqtt is probably redicting stderr. If you want to see all error messages, you can start zigbee2mqtt manually from the directory `/opt/zigbee2mqtt` with `npm start` or `node index.js`

## Introduction

My converter is based on the version currently found in zigbee2mqtt (as of March 25th 2024) and it should hopefully remain backward compatible with the default values of the new settings.

My main concern with the current `PJ-1203A` converter was what should probably called a bug in the device. For channel x, the 4 attributes `energy_flow_x`, `current_x`, `power_x` and `power_factor_x` are regularly provided in that order. This is a bi-directional sensor so `energy_flow_x` describes the direction of the current and can be either `consuming` or `producing`. Unfortunately, that attribute is not properly synchronized with the other three. In practice, that means that zigbee2mqtt reports a incorrect energy flow during a few seconds when swtiching  between `consuming` and `producing`.

For example, suppose that some solar panels are producing an excess of 100W and that a device that consumes 500W is turned on. The following MQTT message would be produced:

    ...
    { "energy_flow_a":"producing" , "power_a":100 , ... } 
    { "energy_flow_a":"producing" , "power_a":400 , ... } 
    { "energy_flow_a":"consuming" , "power_a":400 , ... } 
    ...
    
The second message is obviously incorrect. 

Another issue is that zigbee messages can be lost or reordered. A MQTT client has no way to figure out if the attributes `energy_flow_x`, `power_x`, `current_x` and `power_factor_x` represent a coherent view of a single point in time. Some may be older that others. 

My converter is attempting to solve those issues by delaying the publication of `energy_flow_x`, `power_x`, `current_x` and `power_factor_x` until a complete and hopefully coherent set to attributes is available. It also tries to detect missing or reordered zigbee messages. 

## Variants

Each of the files `PJ_1203A-v*.js` is a variant 

## PJ_1203A-v1.js

This is the first version proposed in this repository. It should be backwark compatible with the 
current converter in zigbee2mqtt but multiple options are provided to fine tune the behavior.

It works but, IMHO it is far too complex. 

## PJ_1203A-v2.js

This is a simplified variant with the following features:
  - a single option to delay the publication until the next update (see above).
  - for each channel, the power, current, power factor and energy flow datapoints 
    are collected and are published together. Nothing is published when some of 
    them are missing or if some zigbee messages were lost or reordered (the 
    collected data may not be coherent).
  - `energy_flow_a` and `energy_flow_b` are not published anymore. Instead `power_a`
    and `power_b` are signed values: positive while consuming and negative 
    while producing.
  - A new attribute `update_x` is publied every time `power_x`, `current_x`, 
    `power_factor_x` are successfully updated on channel `x`. Only the changes 
    to `update_x` are relevant while the actual values are not (e.g. a reset or
    an increase by more than 1 is possible and does NOT indicate that an update 
    was missed). 

## Known issues

My converters have to assume that the device is sending the datapoints in a specific 
order which may not always be true. 

The device probably supports OTA updates so a better firmware may exist somewhere.

Assuming that you have `jq` (https://jqlang.github.io/jq), the relevant information 
can be found in `database.db` with 


```
jq '. | select(.modelId=="TS0601") | {modelId,manufName,appVersion,stackVersion,hwVersion}'  database.db 
```

Otherwise, search `manufName`, `appVersion`, `stackVersion` and `hwVersion` for your TS0601 device.

Currently tested on:
  - `_TZE204_81yrt3lo` with appVersion 74, stackVersion 0 and hwVersion 1
  

