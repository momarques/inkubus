# inkubus

## short description

Inkubus is a ci/cd kubernetes operator designed to watch git branches, pull every new commit from selected branches and run pipeline steps inside kubernetes pods.

## motivation

There's a lot of different approaches to run pipelines these days and all of them have their pros and cons.
In my experience I've run across some of these approaches and I'll try to bring two of them here. Also I just want to create some app to improve my github/CV so people can understand how I understand this tech world.

### ci/cd approach

This one I've used for a long time, and it's very simple:
- use a definition file inside the repository, which can be in yml or whatever else the ci/cd platform uses
- this definition file can have all of the commands which will be run to test, build and deploy the application
- the ci/cd platform can then identify where the file is located and run it
- the steps usually are run in a sequence

One of the things you can normally worry about when using this approach is to decide in what environment the pipeline steps will be run. 

You can run inside the ci/cd platform OS, but you'll have to guarantee that the OS has all the tools needed for every step to happen. Also, it can be a real pain in the *** to deal with file permissions errors.
You can also run inside a docker container, which can be run inside the ci/cd platform, but you'll have to worry about the docker engine installation. Well, actually I think this is a cool approach and there's nothing wrong with that, besides having to take care of the ci/cd platform servers, and you won't be able to scale to many commits in case your team gets bigger unless the platform server is powerful enough to handle it.

### kubernetes approach

Many of the ci/cd platforms have this support, which is basically a deployment of the platform itself inside kubernetes. This can lead us to a cool possibility: run kube pods to execute pipeline jobs. This way all of your tests will be run inside kubernetes and this can help to scale the project branches, i.e., run tests for every commit on every selected branch. The tests will be run inside kubernetes pods so it's easier to scale, since kubernetes have a scaling mechanism which is better than any ci/cd platform may have.
I really like this approach and the idea that I'm building here is actually based on this one, with some differences and references from other tech concepts.

## inkubus approach

The inkubus approach is based on the kubernetes approach and also on GitOps concept, which basically tells us that resources can be pulled from git repositories to the container orchestration platform and be run there. This way we can have a definition for each step and everytime a new commit arrives at the branch, a new pod will be created to run the step with all the specifications we put there. 

## pipeline steps

As I said, each step is defined by a yml file and placed inside a .chart/templates/ directory. It's worth to say that this structure was chosen so it's easier to wrap helm commands into a CLI tool which will add new features to the chart management:
- wrap `helm create` so it can create a full chart directory plus adding yml files used for installing inkubus CRDs
- wrap `helm <test>` so the chart can be verified before being deployed
- wrap `helm package` so the chart can be packaged in a specific version, based on the branch name and commit hash, and uploaded to a local chart repository (maybe a simple persistent storage with chart museum and etcd can do the job) only to store version history
- wrap `helm install` or `helm upgrade` so it can install the chart and change some values with --set

## chart structure

The `helm create` command enable us to initialize a chart with default configurations. We'll not care much about what files are created by default with this command, as it strictly follows the helm chart structure. Instead, we'll watch specific files inside templates/ directory, which represents our CRDs used by inkubus operator.

### values.yml

Some of the configuration values we'll be using on our CRD yaml files will be placed inside values.yml, as a good organization practice. This makes easier to manage the chart values for every kind of resource we'll be using. Also these values may be used to --set the deployment.

### Chart.yaml

This file contains some metadata of your application, like the version, the name, a short description, etc. This will be fulfilled with information from the git repository using --set at the deployment, so for example:
- version: can be represented with the branch name + commit hash or git tag
- name: can be represented with the repository name
- description: can be represented with the repository description
- etc.

### templates/

This is the actual directory used by helm to understand every resource that needs to be created. This is the directory we'll be placing our CRD yml files. 

For every file listed below there's a git connection which is defined as a Watcher. This is actually another pod which will be run specifically on the namespace (branch) you are working and will keep watching the git repository until the pipeline is done. Every time a new commit is tracked by a Watcher, it'll create a pod for each step with commit hash as identification and will execute all the steps related to the commit, sequentially or not. 

In every inkubus CRD file you can define what branches will run each pipeline step, so we can filter 

#### git.yaml

This file defines the git connection:

```yaml
git:
  url: "from values -> github.com/momarques/inkubus-example-app"
  authMethod: "from kube secret > api-token"
  branches:
  - name: "feature/.*"
    steps:
    - "analysis"
    - "test"
  - name: "fix/.*"
    steps:
    - "test"
  - name: "housekeeping/.*"
    steps:
    - "analysis"
    - "test"
  - name: "development"
    steps:
    - "analysis"
    - "test"
    - "vulnerability"
```

This will create Watchers in every new namespace (for each branch) based on the steps selected. 

The Operator will watch for new branches and creates new , while the Watchers will only watch a single git branch


#### test.yaml

This file defines how the tests are going to be executed:

(simple example)
```yaml
test:
  environment:
    baseImage: 
      tag: "from values"
      repo: 
        name: "from values"
        authMethod: "from values > aws-iam"
    scriptFile: "alguma coisa personalizada ou por default na raiz do projeto -> ./test.sh"

```

This will create a `inkubus-test` CRD which will be read by inkubus operator to create pods that will run tests for every selected branch. The context data of every pod will be gathered from the git repository itself, so:
- namespace: based on the name of the branch
- pod id: based om the current commit hash. this will be useful to track what happened, as it'll be sent to any integration the link from a console log output generated by inkubus operator, and also can be checked manually with `kubectl logs`.
The pod will run everything it's on `./test.sh`.

#### analysis.yaml

This file defines how static code analysis will be run:

(simple example)
```yaml
analysis:
  baseImage: 
    tag: "from values"
    repo: 
      name: "from values"
      authMethod: "from values > aws-iam"
  scriptFile: "alguma coisa personalizada ou por default na raiz do projeto -> ./analysis.sh"
  coverage: 
    language: "from values"
    file: "from values"
    cancelThreshold: 65
git:
  url: "from values -> github.com/momarques/inkubus-example-app"
  authMethod: "from kube secret > api-token"
  allowedBranches:
  - "feature/.*"
  - "fix/.*"
  - "housekeeping/.*"
  - "development"
```

This will create a `inkubus-analysis` CRD which will be read by inkubus operator to create pods that will run static code analysis for every selected branch. The coverage section enables us to cancel a pipeline execution if the code coverage % is lower than the threshold defined. Also we need to know the language and the coverage file so we can have a way to check it and extract the %.
The pod will run everything it's on `./analysis.sh`.

#### vulnerability.yaml

This file defines how vulnerability scans will be run:

```yaml
vulnerability:
  baseImage: 
    tag: "from values"
    repo: 
      name: "from values"
      authMethod: "from values > aws-iam"
  scriptFile: "alguma coisa personalizada ou por default na raiz do projeto -> ./vulnerability.sh"
git:
  url: "from values -> github.com/momarques/inkubus-example-app"
  authMethod: "from kube secret > api-token"
  allowedBranches:
  - "feature/.*"
  - "fix/.*"
  - "housekeeping/.*"
  - "development"
  - "main"
```

This will create a `inkubus-vulnerability` CRD which will be read by inkubus operator to create pods that will run vulnerability scans on the code. 
The pod will run everything it's in `./vulnerability.sh`.

#### build.yaml

This file defines how the application build process will be run:

```yaml
build:
  baseImage: 
    tag: "from values"
    repo: 
      name: "from values"
      authMethod: "from values > aws-iam"
  scriptFile: "alguma coisa personalizada ou por default na raiz do projeto -> ./vulnerability.sh"
git:
  url: "from values -> github.com/momarques/inkubus-example-app"
  authMethod: "from kube secret > api-token"
  allowedBranches:
  - "feature/.*"
  - "fix/.*"
  - "housekeeping/.*"
  - "development"
  - "main"
```



#### deliver.yaml

This file defines how the application deliver process will be run:

```yaml
deliver:
  baseImage: 
    tag: "from values"
    repo: 
      name: "from values"
      authMethod: "from values > aws-iam"
  scriptFile: "alguma coisa personalizada ou por default na raiz do projeto -> ./vulnerability.sh"
git:
  url: "from values -> github.com/momarques/inkubus-example-app"
  authMethod: "from kube secret > api-token"
  allowedBranches:
  - "feature/.*"
  - "fix/.*"
  - "housekeeping/.*"
  - "development"
  - "main"
```

## architecture

So how all of this magic is going to happen?
- Inkubus API Operator: made in the operator pattern so it can watch for new CRDs. It's also an API so we can plug a interface later on (CLI tool maybe or js interface)
- Inkubus CRDs: resource definition used by the operator to create resources
- Inkubus Pods: finite pods that will run each desired pipeline step and after finishing, it'll be killed
- Inkubus Watchers: for every CRD created,



## Improvements
- a CRD for each possible integration, with a integration/ directory and <integration_name>.yml definition files, which will integrate the status of every run to external applications, like:
    - github
    - gitlab
    - bitbucket
    - azure git
    - aws git
    - jira
    - slack
    - whatever else, let's develop my people
