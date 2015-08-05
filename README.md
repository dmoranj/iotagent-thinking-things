# iotagent-thinking-things
## Overview
This *Internet of Things Agent* is a bridge can be used to bridge between Telefonica's Thinking Things Closed protocol (TT from now on) and NGSI Context Brokers (like [Orion](https://github.com/telefonicaid/fiware-orion)). The Thinking Things protocol is a simplified protocol aimed to provide a simple platform to experiment with the Internet of Things.

## Usage
In order to install the TT Agent, just clone the project and install the dependencies:
```
git clone https://github.com/dmoranj/iotagent-thinking-things.git
npm install
```
In order to start the IoT Agent, from the root folder of the project, type:
```
bin/thinkingThingsAgent.js
```
## Configuration
All the configuration of the IoT Agent can be customized with the `config.js`. The configuration is divided into to sections: `ngsi` (for all the data concerning the northbound of the Agent) and `thinkingThings` (for all the data concerning the southbound of the Agent).

### Thinking Things configuration
The following parameters can be configured:
* `logLevel`: determines the log level for the incoming TT requests.
* `port`: port where the South Bound will be listening for connections.
* `root`: base path for the South Bound.

### NGSI
* `logLevel`: determines the log level for the connection with the NGSI Broker.
* `plain`: if this flag is true, all the information of every module will be published as a plain attribute in the entity. If the flag is false, an attribute will be created for each module with a compound type, containing an array of all its values (in attribute format).
* `timestamp`: when this flag is true, every attribute will come with a metadata showing the timestamp of its reception in the platform.
* `defaultType`: this is the entity type that will be assigned to the devices whenever there is no other way of determining the appropriate type.
* `contextBroker.host`: host where the NGSI Context Broker is listening.
* `contextBroker.port`: port where the NGSI Context Broker is listening.
* `server.port`: port where the Noth Bound listens for NGSI queries and updates (currently not in used, will be used for commands). 
* `deviceRegistry.type`: the Agent can keep a registry of the connected devices for preconfiguration (e.g.: when the devices come from multiple origins). The possible values are: `mongodb` and `memory`. The memory registry is transient.
* `deviceRegistry.host`: host where the remote database for the device registry is located. This option is not used by memory registries.
* `types`: see Device Configuration below.
* `providerUrl`: public URL and where the IoT Agent is listening for NGSI connections. This URL only makes sense for its use with commands and passive attributes (read and write).
* `deviceRegistrationDuration`: lifetime of the registration of the IoT Agent as a Context Provider for active attributes of the device entities.

### Device configuration
There are two ways for a device to be used with the IoT Agent. 

#### One IoT Agent per Group: no preregistration needed
The first and most simple way is to use one IoT Agent per group of devices, configuring the service and subservice information (and security information when appliable) in the default type. In this mode, the `types` attribute of the NGSI configuration have to contain all the service information. Whenever a new request arrives to the Agent coming from the South Bound, the default type will be assigned to the device, and the configured information of that type applied to the device registry (and used in further communications).

#### One Global IoT Agent: preregistration of the devices needed
If there is a single global agent, all the devices must be preregistered using the provisioning API (that is listening in the `server.port` port in the `/iot/devices` path). The device information can be sent to this API, and written to the device registry, thus configuring the service information in a per device basis.

To configure a type, a new attribute with the type name should be added to the `ngsi.types` object with the name of the new type in the `config.js` file:
```
config.ngsi = {

	[...]

    types: {
        'ThinkingThing': {
            service: 'smartGondor',
            subservice: '/gardens',
            commands: [],
            lazy: [],
            active: [
                {
                    name: 'humidity',
                    type: 'Number'
                }
            ]
        }
    },

	[...]

};
```

The format for new device registrations is as follows:
```
{
    "name": "Light1",
    "service" : "smartGondor",
    "service_path": "/gardens",
    "entity_name": "TheFirstLight",
    "entity_type": "TheLightType",
    "attributes": [],
    "commands": [
        {
            "name": "luminance",
            "type": "lumens"
        }
    ]
}
```
## Thinking Things Protocol
### Overview
The thinking things protocol offer a lightweight HTTP-based protocol aimed to constrained devices who whishes to communicate
with backends, sending simple sensor data and receiving simple configuration parameters and commands. This protocol was designed
as a part of [Telefonica Thinking Things](http://www.thinkingthings.telefonica.com/) project.

### Protocol basics
All protocol interactions are started from the client device. The device sends an HTTP POST request to the server with Content-Type
`application/x-www-form-urlencoded`, containing a single field named `cadena`, with a payload that looks like the following example:

```
#ITgAY,#0,P1,214,07,b00,444,-47,#0,K1,300$,#3,B,4.70,1,1,1,1,0,-1$#4,T1,31.48,0$#4,H1,31.48,1890512.00,0$#4,LU,142.86,0$
```
The first value, corresponding to the value between the first two '#' characters is the Stack ID, i.e.: the ID of the device
itself.

This payload can be divided in modules, each one of them responsible for a single measure. Modules are separated by the 
'#' character, and all of them consists of a series of parameters sepparated by commas. The first parameter is always 
interpreted as the ID of the module. The second parameter identifies the kind of module (that will decide the interpretation
of the rest of the parameters), and the rest of the values will be interpreted based on the module type.

Let's look at an example more closely, extracting it from the payload above: the module `#4,H1,31.48,1890512.00,0$`. This
module has:
- An id: 4
- A module type: H1. This means the module is a Humidity sensor.
- Two values. Knowing that the module is a Humidity sensor, we know that this values can be interpreted as the temperature
and the humidity values (that can be used to calculate real humidity).
- An optional sleep value all the TT modules should implement (not used in this case, thus the value 0$).

The following subsection shows all the available modules.

### Modules
#### GM Generic Module
This module can be used to send arbitrary attribute information to the server. Each GM can be used to send a single 
attribute with the attribute name specified in the module parameters.

| Position | Meaning                      | Example             |
|:--------:|:---------------------------- |:------------------- |
| 1        | Attribute name               | pressure            |
| 2        | Attribute value              | 790                 |
| 3        | Sleeping value (unused)      | 0$                  |

#### GC Generic Configuration
This module represents a generic configuration parameter, that will be stored by the server. Each time the device sends
a GC module to the server, the server will reply with the last available value to the client.

| Position | Meaning                      | Example             |
|:--------:|:---------------------------- |:------------------- |
| 1        | Parameter name               | timeout             |
| 2        | Parameter value              | 2000                |
| 3        | Sleeping value (unused)      | 0$                  |


#### K1 Core Module
This module is mandatory for all the payloads sent to the server. This module represents the Core communication module, 
and can be used to configure the sleeping time of the device.

#### H1
Sends information about the temperature and humidity. The values given by the sensor are usually raw vales (in the 
Thinking Things Closed devices at least), so some processing may be needed before using the values.

Parameters:

| Position | Meaning                      | Example             |
|:--------:|:---------------------------- |:------------------- |
| 1        | Temperature                  | 26.29               |
| 2        | Humidity                     | 1890512.00          |
| 3        | Sleeping value (unused)      | 0$                  |
 
#### LU Luminance
Sends information about luminance.

| Position | Meaning                      | Example             |
|:--------:|:---------------------------- |:------------------- |
| 1        | Luminance                    | 142.86              |
| 2        | Sleeping value (unused)      | 0$                  |

#### GPS Coordinates
Sends the GPS coordinates to the server.

| Position | Meaning                      | Example             |
|:--------:|:---------------------------- |:------------------- |
| 1        | Latitude                     | 21.1                |
| 2        | Longitude                    | -9.4                |
| 3        | Speed                        | 12.3                |
| 4        | Orientation                  | 0.64                |
| 5        | Altitude                     | 127                 |
| 6        | Sleeping value (unused)      | 0$                  |

#### P1
Sends information about the GSM connection to the server.

| Position | Meaning                      | Example             |
|:--------:|:---------------------------- |:------------------- |
| 1        | mcc                          | 384                 |
| 2        | mnc                          | 09                  |
| 3        | lac                          | a01                 |
| 4        | cellid                       | 434                 |
| 5        | dbm                          | -57                 |
| 6        | Sleeping value (unused)      | 0$                  |

#### T1
Sends temperature information to the server.

| Position | Meaning                      | Example             |
|:--------:|:---------------------------- |:------------------- |
| 1        | Temperature                  | 22.86               |
| 2        | Sleeping value (unused)      | 0$                  |

#### B
Sends battery information to the server.

| Position | Meaning                      | Example             |
|:--------:|:---------------------------- |:------------------- |
| 1        | Voltage                      | 4.70                |
| 2        | State                        | 1                   |
| 3        | Charger                      | 1                   |
| 4        | Charging                     | 1                   |
| 5        | Mode                         | 1                   |
| 6        | Desconnection                | 0                   |
| 2        | Sleeping value (unused)      | 0$                  |

## Client
In order to test the IoT Agent, a ThinkingThings client is provided that can emulate some calls from TT devices. The client can be started from the root folder of the project with the following command:
```
bin/thinkingThingsClient
```
The client provides a prompt with several commands that let you send information like different modules, configure the remote server or prepare module stacks to send multiple measures at once. In order to show all the available commands, from the client prompt, type:
```
help
```
### Stacks
In standard mode, each module command send an HTTP request to the remote server with the module information (and the prefixed stack id). In order to send a stack of modules, a stack must be made. 

A stack can be started with the followign command:
```
startStack
```
All the subsequent calls to measure modules will not send the module information, but will stack it instead. In order to clean the stack and send all the stacked information to the remote server type:
```
sendStack
```

### Commands
```
humidity <temperature> <pressure> <moduleId>  

	Send a new humidity measure

gps <latitude> <longitude> <speed> <orientation> <altitude> <moduleId>  

	Send a new gps measure

temperature <temperature> <moduleId>  

	Send a new temperature measure

setConfig <host> <port> <path> <stackId>  

	Change the configuration of the device.

getConfig  

	Read the current configuration.

setSleep <value> <condition>  

	Set the default sleep parameters.

getSleep  

	Get the current default sleep parameters.

startStack  

	Start stacking the payloads to send a multimodule payload (stackMode = off).

sendStack  

	Send all the stacked module info (stackMode = on).
```

## Development documentation
### Project build
The project is managed using Grunt Task Runner.

For a list of available task, type
```bash
grunt --help
```

The following sections show the available options in detail.


### Testing
[Mocha](http://visionmedia.github.io/mocha/) Test Runner + [Chai](http://chaijs.com/) Assertion Library + [Sinon](http://sinonjs.org/) Spies, stubs.

The test environment is preconfigured to run [BDD](http://chaijs.com/api/bdd/) testing style with
`chai.expect` and `chai.should()` available globally while executing tests, as well as the [Sinon-Chai](http://chaijs.com/plugins/sinon-chai) plugin.

Module mocking during testing can be done with [proxyquire](https://github.com/thlorenz/proxyquire)

To run tests, type
```bash
grunt test
```

Tests reports can be used together with Jenkins to monitor project quality metrics by means of TAP or XUnit plugins.
To generate TAP report in `report/test/unit_tests.tap`, type
```bash
grunt test-report
```


### Coding guidelines
jshint, gjslint

Uses provided .jshintrc and .gjslintrc flag files. The latter requires Python and its use can be disabled
while creating the project skeleton with grunt-init.
To check source code style, type
```bash
grunt lint
```

Checkstyle reports can be used together with Jenkins to monitor project quality metrics by means of Checkstyle
and Violations plugins.
To generate Checkstyle and JSLint reports under `report/lint/`, type
```bash
grunt lint-report
```


### Continuous testing

Support for continuous testing by modifying a src file or a test.
For continuous testing, type
```bash
grunt watch
```


### Source Code documentation
dox-foundation

Generates HTML documentation under `site/doc/`. It can be used together with jenkins by means of DocLinks plugin.
For compiling source code documentation, type
```bash
grunt doc
```


### Code Coverage
Istanbul

Analizes the code coverage of your tests.

To generate an HTML coverage report under `site/coverage/` and to print out a summary, type
```bash
# Use git-bash on Windows
grunt coverage
```

To generate a Cobertura report in `report/coverage/cobertura-coverage.xml` that can be used together with Jenkins to
monitor project quality metrics by means of Cobertura plugin, type
```bash
# Use git-bash on Windows
grunt coverage-report
```


### Code complexity
Plato

Analizes code complexity using Plato and stores the report under `site/report/`. It can be used together with jenkins
by means of DocLinks plugin.
For complexity report, type
```bash
grunt complexity
```

### PLC

Update the contributors for the project
```bash
grunt contributors
```


### Development environment

Initialize your environment with git hooks.
```bash
grunt init-dev-env 
```

We strongly suggest you to make an automatic execution of this task for every developer simply by adding the following
lines to your `package.json`
```
{
  "scripts": {
     "postinstall": "grunt init-dev-env"
  }
}
``` 


### Site generation

There is a grunt task to generate the GitHub pages of the project, publishing also coverage, complexity and JSDocs pages.
In order to initialize the GitHub pages, use:

```bash
grunt init-pages
```

This will also create a site folder under the root of your repository. This site folder is detached from your repository's
history, and associated to the gh-pages branch, created for publishing. This initialization action should be done only
once in the project history. Once the site has been initialized, publish with the following command:

```bash
grunt site
```

This command will only work after the developer has executed init-dev-env (that's the goal that will create the detached site).

This command will also launch the coverage, doc and complexity task (see in the above sections).

