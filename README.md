# Lighter
[Lighter](https://en.wikipedia.org/wiki/Lighter_(barge)) solves the problem of automating 
deployments to [Marathon](https://github.com/mesosphere/marathon) and handling of differences
between multiple environments. Given a heirachy of yaml files and environments Ligther can 
expand service config files and deploy them to Marathon. 

For even tighter integration into the development process Lighter can resolve Marathon config files 
from Maven and merge these with environment specific overrides. This enables continuous deployment 
whenever new releases or snapshots appear in the Maven repository. Optional version range constraints 
allows for example patches/minor versions to be rolled out continuously, while requiring a config
change to roll out major versions.

## Environment Variables

 * **MARATHON_URL** - Marathon url, e.g. "http://marathon-host:8080/"

## Usage

```
usage: lighter COMMAND [OPTIONS]...

Marathon deployment tool

positional arguments:
  {deploy,verify}  Available commands
    deploy         Deploy services to Marathon
    verify         Verify and generate Marathon configuration files

optional arguments:
  -h, --help       show this help message and exit
  -n, --noop       Execute dry-run without modifying Marathon [default: False]
  -v, --verbose    Increase logging verbosity [default: False]
```

### Deploy Command

```
usage: lighter deploy [OPTIONS]... YMLFILE...

Deploy services to Marathon

positional arguments:
  YMLFILE               Service files to expand and deploy

optional arguments:
  -h, --help            show this help message and exit
  -m MARATHON, --marathon MARATHON
                        Marathon url, e.g. "http://marathon-host:8080/"
  -f, --force           Force deployment even if the service is already
                        affected by a running deployment [default: False]
```

## Configuration
Given a directory structure like

```
my-config-repo/
|   globals.yml
└─ production/
|   |   globals.yml
|   └─ services/
|          myservice.yml
|          myservice2.yml
└─ staging/
    |   globals.yml
    └─ services/
           myservice.yml
```

Running `lighter deploy -m http://marathon-host:8080 staging/services/myservice.yml` will

* Merge *myservice.yml* with environment defaults from *my-config-repo/staging/globals.yml* and *my-config-repo/globals.yml*
* Fetch the *json* template for this service and version from the Maven repository
* Expand the *json* template with variables and overrides from the *yml* files
* Post the resulting *json* configuration into Marathon

### Maven
The `maven:` section specifies where to fetch *json* templates from which are 
merged into the configuration. For example

*globals.yml*
```
maven:
  repository: "http://username:password@maven.example.com/nexus/content/groups/public"
```

*myservice.yml*
```
maven:
  groupid: 'com.example'
  artifactid: 'myservice'
  version: '1.0.0'
  classifier: 'marathon'
```

The Maven 'classifier' tag is optional.

#### Dynamic Versions
Versions can be dynamically resolved from Maven using a range syntax.

```
maven:
  groupid: 'com.example'
  artifactid: 'myservice'
  resolve: '[1.0.0,2.0.0)'
```

For example

Expression | Resolve To
:----------|:-----------
[1.0.0,2.0.0) | 1.0.0 up to but not including 2.0.0
[1.0.0,1.2.0] | 1.0.0 up to and including 1.2.0
[1.0.0,2.0.0)-featurebranch | 1.0.0 up to and including 1.2.0, only matches *featurebranch* releases
[1.0.0,1.2.0]-SNAPSHOT | 1.0.0 up to and including 1.2.0, only matches *SNAPSHOT* versions
[1.0.0,2.0.0]-featurebranch-SNAPSHOT | 1.0.0 up to and including 1.2.0, only matches *featurebranch-SNAPSHOT* versions
[1.0.0,] | 1.0.0 or greater
(1.0.0,] | Greater than 1.0.0
[,] | Latest release version
[,]-SNAPSHOT | Latest *SNAPSHOT* version

##### Snapshot Builds
If an image is rebuilt with the same Docker tag Marathon won't detect a change and roll out the new image. To ensure
that new snapshot/latest versions are deployed use `%{lighter.uniqueVersion}` and `forcePullImage` like for example

*myservice.yml*
```
override:
  container:
    docker:
      forcePullImage: true
  env:
    SERVICE_BUILD: '%{lighter.uniqueVersion}'
```

### Freestyle Services
Yaml files may contain a `service:` tag which specifies a Marathon *json* fragment 
to use as the service configuration base for further merging. This allows for
services which aren't based on a *json* template but rather defined exclusively 
in *yaml*.

*myservice.yml*
```
service:
  id: '/myproduct/myservice'
  container:
    docker:
      image: 'meltwater/myservice:latest'
  env:
    DATABASE: 'database:3306'
  cpus: 1.0
  mem: 1200
  instances: 1
```

### Overrides
Yaml files may contain an `override:` section that will be merged directly into the Marathon json. The 
structure contained in the `override:` section must correspond to the [Marathon REST API](https://mesosphere.github.io/marathon/docs/rest-api.html#post-v2-apps). For example 

```
override:
  instances: 4
  cpus: 2.0
  env:
    LOGLEVEL: 'info'
    NEW_RELIC_APP_NAME: 'MyService Staging'
    NEW_RELIC_LICENSE_KEY: '123abc'
```

### HipChat
Yaml files may contain an `hipchat:` section that specifies where to announce deployments. Create a [HipChat V2 token](https://www.hipchat.com/docs/apiv2) that is allowed to post to rooms. For example

```
hipchat:
  token: '123abc'
  rooms:
    - '123456'
```

### NewRelic
If [NewRelic deployment notifications](https://docs.newrelic.com/docs/apm/new-relic-apm/maintenance/deployment-notifications) are desired you need to 
both supply your [NewRelic REST API key](https://docs.newrelic.com/docs/apis/rest-api-v2/requirements/api-keys) (NOT the license key given to the agent) and set the `NEW_RELIC_APP_NAME` environment variable on the service:

*globals.yml*
```
newrelic:
  token: 'i_am_an_api_token'
```

*myservice.yml*
```
override:
  env:
    NEW_RELIC_APP_NAME: 'MyService'
```

You might want to use prefixes per environment:
*staging/globals.yml*
```
variables:
  newrelic.appname.prefix: 'Staging'
```

*staging/production.yml*
```
variables:
  newrelic.appname.prefix: 'Prod'
```

*myservice.yml*
```
override:
  env:
    NEW_RELIC_APP_NAME: '%{newrelic.appname.prefix} MyService'
```

### Variables
Yaml files may contain an `variables:` section containing key/value pairs that will be substituted into the *json* template. All 
variables in a templates must be resolved or it's considered an error. This can be used to ensure that some parameters are 
guaranteed to be provided to a service. For example
```
variables:
  docker.registry: 'docker.example.com'
  rabbitmq.host: 'rabbitmq-hostname'
```

And used from the *json* template like
```
{
    "id": "/myproduct/myservice",
    "container": {
        "docker": {
            "image": "%{docker.registry}/myservice:1.2.3"
        }
    },
    "env": {
        "rabbitmq.url": "amqp://guest:guest@%{rabbitmq.host}:5672"
    }
}
```

#### Predefined Variables

Variable | Contains
:--------|:--------
%{lighter.version} | Fixed or resolved Maven version
%{lighter.uniqueVersion} | Unique build version resolved from Maven metadata

### Facts
Yaml files may contain a `facts:` section with information about the service surroundings

```
facts:
  environment: 'staging'
```

## Deployment
Place a `lighter` script in the root of your configuration repo.

```
#!/bin/sh
set -e

# Must mount this directory into the container so that all globals.yml are found
OLDCWD="`pwd`"
cd "`dirname $0`"
NEWCWD="`pwd`"

if [ "$OLDCWD" != "$NEWCWD" ]; then
  LIGHTER_DOCKER_OPTS="$LIGHTER_DOCKER_OPTS -v ${NEWCWD}:${NEWCWD} -v ${OLDCWD}:${OLDCWD} --workdir ${OLDCWD}"
else
  LIGHTER_DOCKER_OPTS="$LIGHTER_DOCKER_OPTS -v ${NEWCWD}:${NEWCWD} --workdir ${NEWCWD}"
fi

# Ligher will write the expanded json files to ./output 
exec docker run --rm --net=host -v "${NEWCWD}/output:/tmp/lighter" $LIGHTER_DOCKER_OPTS meltwater/lighter:latest $@
```

Execute the script for example like

```
cd my-config-repo

# Make sure you're using the latest lighter version (if you're not using a fixed version)
docker pull meltwater/lighter:latest

# Deploy/sync all services (for example from Jenkins or other CI/CD server)
./lighter deploy -f -m http://marathon-host:8080 $(find staging -name \*.yml -not -name globals.yml)

# Deploy single services
./lighter deploy -m http://marathon-host:8080 staging/myservice.yml staging/myservice2.yml
```
