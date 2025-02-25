# GPIO Device Service
[![Build Status](https://jenkins.edgexfoundry.org/view/EdgeX%20Foundry%20Project/job/edgexfoundry/job/device-gpio/job/main/badge/icon)](https://jenkins.edgexfoundry.org/view/EdgeX%20Foundry%20Project/job/edgexfoundry/job/device-gpio/job/main/) [![Go Report Card](https://goreportcard.com/badge/github.com/edgexfoundry/device-gpio)](https://goreportcard.com/report/github.com/edgexfoundry/device-gpio) [![GitHub Latest Dev Tag)](https://img.shields.io/github/v/tag/edgexfoundry/device-gpio?include_prereleases&sort=semver&label=latest-dev)](https://github.com/edgexfoundry/device-gpio/tags) ![GitHub Latest Stable Tag)](https://img.shields.io/github/v/tag/edgexfoundry/device-gpio?sort=semver&label=latest-stable) [![GitHub License](https://img.shields.io/github/license/edgexfoundry/device-gpio)](https://choosealicense.com/licenses/apache-2.0/) ![GitHub go.mod Go version](https://img.shields.io/github/go-mod/go-version/edgexfoundry/device-gpio) [![GitHub Pull Requests](https://img.shields.io/github/issues-pr-raw/edgexfoundry/device-gpio)](https://github.com/edgexfoundry/device-gpio/pulls) [![GitHub Contributors](https://img.shields.io/github/contributors/edgexfoundry/device-gpio)](https://github.com/edgexfoundry/device-gpio/contributors) [![GitHub Committers](https://img.shields.io/badge/team-committers-green)](https://github.com/orgs/edgexfoundry/teams/device-gpio-committers/members) [![GitHub Commit Activity](https://img.shields.io/github/commit-activity/m/edgexfoundry/device-gpio)](https://github.com/edgexfoundry/device-gpio/commits)

## Overview
GPIO Micro Service - device service for connecting GPIO devices to EdgeX

- Function:
  - This device service uses sysfs ABI (default) or chardev ABI (experiment) to control GPIO devices
  For a connected GPIO device, update the configuration files, and then start to read or write data from GPIO device
- This device service **ONLY works on Linux system**
- This device service is contributed by [Jiangxing Intelligence](https://www.jiangxingai.com)

## Build with NATS Messaging
Currently, the NATS Messaging capability (NATS MessageBus) is opt-in at build time.
This means that the published Docker image and Snaps do not include the NATS messaging capability.

The following make commands will build the local binary or local Docker image with NATS messaging
capability included.
```makefile
make build-nats
make docker-nats
```

The locally built Docker image can then be used in place of the published Docker image in your compose file.
See [Compose Builder](https://github.com/edgexfoundry/edgex-compose/tree/main/compose-builder#gen) `nat-bus` option to generate compose file for NATS and local dev images.

## Packaging

This component is packaged as docker image and snap.

For docker, please refer to the [Dockerfile] and [Docker Compose Builder] scripts.

For the snap, refer to the [snap] directory.

[Dockerfile]: Dockerfile
[Docker Compose Builder]: https://github.com/edgexfoundry/edgex-compose/tree/main/compose-builder
[snap]: snap

## Usage
- This Device Service runs with other EdgeX Core Services, such as Core Metadata, Core Data, and Core Command
- The gpio device service can contains many pre-defined devices which were defined by [device.custom.gpio.toml](cmd/res/devices/device.custom.gpio.toml) such as `GPIO-Device01`. These devices are created by the GPIO device service in core metadata when the service first initializes
- Device profiles ([device.custom.gpio.yaml](cmd/res/profiles/device.custom.gpio.yaml)) are used to describe the actual GPIO hardware of a device and allow individual gpios to be given human-readable names/aliases
- After the gpio device service has started, we can read or write these corresponding pre-defined devices

```yaml
name: "Custom-GPIO-Device"
manufacturer: "Jiangxing Intelligence"
model: "SP-01"
labels:
  - "device-custom-gpio"
description: "Example of custom gpio device"

deviceResources:
  -
    name: "Power"
    isHidden: false
    description: "mocking power button"
    attributes: { line: 17 }
    properties:
      valueType: "Bool"
      readWrite: "RW"

  -
    name: "LED"
    isHidden: false
    description: "mocking LED"
    attributes: { line: 27 }
    properties:
      valueType: "Bool"
      readWrite: "W"

  -
    name: "Switch"
    isHidden: false
    description: "mocking switch"
    attributes: { line: 22 }
    properties:
      valueType: "Bool"
      readWrite: "R"
```

- Since GPIO sysfs interface is **deprecated after Linux version 4.8**, we provide two ABI interfaces: the sysfs version and the new chardev version. By default we set interface to sysfs, and you can change it inside `[DeviceList.Protocols.interface]` section of `configuration.toml`. For the chardev interface, you still need to specify a selected chip, this is also under `[DeviceList.Protocols.interface]` section.

## Guidance
Here we give two step by step guidance examples of using this device service. In these examples, we use RESTful API to interact with EdgeX (please notice that, you still need to use Core Command service rather than directly interact with GPIO device service).

Since the `edgex-cli` has released, we can use this new approach to operate devices:

`edgex-cli command list -d GPIO-Device01`

If you would prefer the traditional RESTful way to operate, you can try:

`curl http://localhost:59882/api/v3/device/name/GPIO-Device01`

Use the `curl` response to get the command URLs (with device and command ids) to issue commands to the GPIO device via the command service as shown below. You can also use a tool like `Postman` instead of `curl` to issue the same commands.

```json
{
    "apiVersion": "v2",
    "statusCode": 200,
    "deviceCoreCommand": {
        "deviceName": "GPIO-Device01",
        "profileName": "Custom-GPIO-Device",
        "coreCommands": [
            {
                "name": "Power",
                "get": true,
                "set": true,
                "path": "/api/v3/device/name/GPIO-Device01/Power",
                "url": "http://edgex-core-command:59882",
                "parameters": [
                    {
                        "resourceName": "Power",
                        "valueType": "Bool"
                    }
                ]
            },
            {
                "name": "LED",
                "set": true,
                "path": "/api/v3/device/name/GPIO-Device01/LED",
                "url": "http://edgex-core-command:59882",
                "parameters": [
                    {
                        "resourceName": "LED",
                        "valueType": "Bool"
                    }
                ]
            },
            {
                "name": "Switch",
                "get": true,
                "path": "/api/v3/device/name/GPIO-Device01/Switch",
                "url": "http://edgex-core-command:59882",
                "parameters": [
                    {
                        "resourceName": "Switch",
                        "valueType": "Bool"
                    }
                ]
            }
        ]
    }
}
```

### Direction setting with sysfs
When using sysfs, the operations to access and "read" or "write" the GPIO pins are to:

1. Export the pin
2. Set the direction (either IN or OUT)
3. Read the pin input or write the pin value based on the direction
4. Unexport the pin 

When using sysfs, setting the direction causes the value to be reset.  Therefore, this implementation only sets the direction on opening the line to the GPIO.  After that, it is assumed the same direction is used while the pin is in use and exported.

The direction is set by an optional attribute in the device profile called `defaultDirection`.  It can be set to either "in" or "out".  If it is not set, the default direction is assumed to be "out".

``` yaml
  -
    name: "LED"
    isHidden: false
    description: "mocking LED"
    attributes: { line: 27, defaultDirection: "out" }
    properties:
      valueType: "Bool"
      readWrite: "W"
```

Note:  the direction should not be confused with the device profile's read/write property.  If you set the defaultDirection to "in" but then set the readWrite property to "RW" or "W", any attempt to write to the pin will result in a "permission denied" error.  For consistency sake, when your defaultDirection is "in" set readWrite to "R" only.

### Write value to GPIO
Assume we have a GPIO device (used for power enable) connected to gpio17 on current system of raspberry pi 4b. When we write a value to GPIO, this gpio will give a high voltage.

```shell
# Set the 'Power' gpio to high
$ curl -X PUT -d   '{"Power":"true"}' http://localhost:59882/api/v3/device/name/GPIO-Device01/Power
{"apiVersion":"v2","statusCode":200}
$ cat /sys/class/gpio/gpio17/direction ; cat /sys/class/gpio/gpio17/value
out
1

# Set the 'Power' gpio to low
$ curl -X PUT -d   '{"Power":"false"}' http://localhost:59882/api/v3/device/name/GPIO-Device01/Power
{"apiVersion":"v2","statusCode":200}
$ cat /sys/class/gpio/gpio17/direction ; cat /sys/class/gpio/gpio17/value
out
0
```

Now if you test gpio17 of raspberry pi 4b , it is outputting high voltage.


### Read value from GPIO
Assume we have another GPIO device (used for button detection) connected to pin 22 on current system. When we read a value from GPIO, this gpio will be exported and set direction to input.

```shell
$ curl http://localhost:59882/api/v3/device/name/GPIO-Device01/Switch
```

Here, we post some results:

```bash
{
  "apiVersion": "v2",
  "statusCode": 200,
  "event": {
    "apiVersion": "v2",
    "id": "a6104256-92a4-41a8-952a-396cd3dabe25",
    "deviceName": "GPIO-Device01",
    "profileName": "Custom-GPIO-Device",
    "sourceName": "Switch",
    "origin": 1634221479227566300,
    "readings": [
      {
        "id": "240dc2ea-d69f-4229-94c4-3ad0507cf657",
        "origin": 1634221479227566300,
        "deviceName": "GPIO-Device01",
        "resourceName": "Switch",
        "profileName": "Custom-GPIO-Device",
        "valueType": "Bool",
        "value": "false"
      }
    ]
  }
}
```



### docker-compose.yml 

Add the `device-gpio` to the docker-compose.yml of edgex foundry 2.0-Ireland.

```yml
...
    device-gpio:
        container_name: edgex-device-gpio
        depends_on:
        - consul
        - data
        - metadata
        environment:
          CLIENTS_CORE_COMMAND_HOST: edgex-core-command
          CLIENTS_CORE_DATA_HOST: edgex-core-data
          CLIENTS_CORE_METADATA_HOST: edgex-core-metadata
          CLIENTS_SUPPORT_NOTIFICATIONS_HOST: edgex-support-notifications
          CLIENTS_SUPPORT_SCHEDULER_HOST: edgex-support-scheduler
          DATABASES_PRIMARY_HOST: edgex-redis
          EDGEX_SECURITY_SECRET_STORE: "false"
          MESSAGEQUEUE_HOST: edgex-redis
          REGISTRY_HOST: edgex-core-consul
          SERVICE_HOST: edgex-device-gpio
        hostname: edgex-device-gpio
        image: edgexfoundry/device-gpio:0.0.0-dev
        networks:
          edgex-network: {}
        ports:
        - 59910:59910/tcp
        read_only: false
        privileged: true
        volumes:
        - "/sys:/sys"
        - "/dev:/dev"
        security_opt:
        - no-new-privileges:false
        user: root:root
...
```



## API Reference

- `device-custom-gpio-profile.yaml`

  | Core Command | Method | parameters        | Description                                                  | Response |
  | ------------ | ------ | ----------------- | ------------------------------------------------------------ | -------- |
  | Power        | put    | {"Power":<value>} | Set value for the specified gpio<br/><value>: bool, "true" or "false" | 200 ok   |
  |              | get    |                   | Get value of the specified gpio<br/>valueType: Bool          | "true"   |



## License
[Apache-2.0](LICENSE)

