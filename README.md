# kabanero-events
[![Build Status](https://travis-ci.org/kabanero-io/kabanero-events.svg?branch=master)](https://travis-ci.org/kabanero-io/kabanero-events)

## Table of Contents
* [Introduction](#Introduction)   
* [Sample Event Trigger](#Sample_Trigger)   
* [Functional Specification](#Functional_Spec)   
* [Building](#Building)   

<a name="Introduction"></a>
## Introduction 

This repository contains the experimental webhook and events components of Kabanero.

**NOTE: The webhook and events components are experimental, and the specification may be updated with incompatible changes in the next few Kabanero releases.**

**Note: The webhook portion of this specification shall be moved to a separate repository so there are distinct web hook and event components.**

<a name="Sample_Trigger"></a>
## Sample Event Trigger

The incubator collection shipped with Kabanero 0.4 contains a sample event trigger that demonstrates:
- Organizational webhook
- Triggering of 3 different pipelines based on github events:
  - A build only pipeline for a pull request to master branch
  - A build and push pipeline when pushing to master. 
  - A retag pipeline for tagging an image based on a tag to Github.

### Configurations

#### Enabling the webhook listener

#### Configuring NATS

#### Tekton Configuration

Configure Github **basic authentication** for Tekton builds by following the instructions here: https://github.com/tektoncd/pipeline/blob/master/docs/auth.md.

Note: Basic authentication is the minimum required configuration to authenticate with Github. It is used to retrieve individual configuration files from Github to determine the type of repository being built, without first cloning the repository. It can also be used to clone the repository.  

In addition to basic authentication, you may also optionally configure SSH to clone the repository prior to builds. The first step is to use the SSH URL for a GitHub repository in your PipelineResource for your git source. To use ssh URL, update the `build.repoURL` in `eventTriggers.yaml` to use `message.webhook.body.repository.clone_url`.

Another necessary step is to base64 encode the SSH private key associated with your GitHub account and add it to a
secret. An example Secret has been provided below.

#### ssh-key-secret.yaml
```yaml
 apiVersion: v1
 kind: Secret
 metadata:
   name: ssh-key
   annotations:
     tekton.dev/git-0: github.ibm.com # URL to GH or your GHE
 type: kubernetes.io/ssh-auth
 data:
   ssh-privatekey: <base64 encoded private key> # This can be generated using `cat ~/.ssh/id_rsa | base64`

   # This is non-standard, but its use is encouraged to make this more secure.
   #known_hosts: <base64 encoded>
```

After applying the Secret with `kubectl apply -f ssh-key-secret.yaml`, associate it with the ServiceAccount you created to run the Appsody Tekton builds. For example, this can be done with `kabanero-operator` service account, which is used by the samples by running
```bash
$ kubectl edit sa kabanero-operator yaml
```

and then appending `ssh-key` to the `secrets` section like so:
```
 apiVersion: v1
 imagePullSecrets:
 - name: kabanero-operator-dockercfg-4vzbk
 kind: ServiceAccount
 metadata:
   creationTimestamp: "2019-10-22T14:05:09Z"
   name: kabanero-operator
   namespace: kabanero
 secrets:
 - name: kabanero-operator-token-r7kdg
 - name: kabanero-operator-dockercfg-4vzbk
 - name: ssh-key
```

#### Github configuration

Create an organization on Github.

Configure an organizational webhook following the instruction here: https://help.github.com/en/github/setting-up-and-managing-your-enterprise-account/configuring-webhooks-for-organization-events-in-your-enterprise-account.
- For `payload URl`, enter the route to your webhook, such as `kabanero-events-kabaner.<host>.com`. The actual URL is installation dependent.
- For `Content type` select `application/json`
- For `Secret`, leave blank for now, as the current implementation does not yet support checking secrets.
- For the list of events, select `send me everything`.

### Running the Sample

#### Initial Push to master

- Create a new empty repository for the org, say `project1`.  Note the url for the new repository.
- On the developer machine, initialize the appsody project:
```
mkdir  project1
cd project1
appsody init nodejs-express
git init
git remote add origin <url>
git remote add origin git@<host>:<owner>/project1.git
git add * .appsody-config.yaml .gitignore
git commit -m "initial drop"
git push -u origin master
```
- Check your Tekton dashboard for a new build that extracts from master, and pushes the result to the internal openshift registry as: kabanero/project1:sha.

#### Build from a pull request

- Create a branch on the repository
- Push some code to the branch
- Create a pull request
- Check your Tekton dashboard for a new build that just performs a test build.

#### Creating a new version via Github Tagging

- Create a new namespace project1-test
- Switch back go the master repository
- Use `git log` to locate the SHA of the commits. Pick a commit to be tag.
- Tag the commit as version 0.0.1:
```
git tag 0.0.1 SHA
git push orig 0.0.1
```
- Check the Tekton dashboard for a new build that tags the original docker image at kabanero:project1:SHA into a new image project1-test:project1:0.0.1 and pushed into the new namespace.


<a name="Functional_Spec"></a>
## Functional Specifications


### Definitions

A Kabanero installations is an installation of a specific version of the Kabanero operator via the operator installer.

A Kabanero instance is an instance of the Kabanero runtime created by applying custom resource of kind `kabanero` to a Kabanero installation.

The webhook component of Kabanero receives webhook POST events through its listener. The webhook listener is created when a Kabanero instance is created with kabanero events component enabled. Its primary purpose is to publish webhook events to a message destination.

The events component of Kabanero is designed to mediate and choreograph the operations of other components within Kabanero via events. It may be used to filter and transform events, and initiate additional actions based on events.


# Message Providers and Destinations

The available message providers and their destinations are stored in `eventDefinitions.yaml`. The format for message providers is:

```
messageProviders:
- name: <name of provider>
  providerType: nats | rest
  url: <url of provider>
  timeout: <timeout to send/receive message>
```

The supported provider types are:
- nats: a NATS provider
- rest: a REST endpoint provider that only allows sending a message

The format for event destinations is:
```
eventDestinations:
- name: <name of destination>
  providerRef: <name of provider>
  topic: <name of topic>
  skipTLSVerify: true | false
```

Below is a sample eventDefinitions.yaml file :
```
messageProviders:
- name: nats-provider
  providerType: nats
  url: nats://127.0.0.1:4222
  timeout: 1h
- name: webhook-site-provider
  providerType: rest
  url: https://webhook.site/5d82669e68c
- name: tekton-provider
  providerType: rest
  url: https://event-listener.scolded.fyre.ibm.com:8080
  skipTLSVerify: true
eventDestinations:
- name: github
  providerRef: nats-provider
  topic: github
- name: passthrough-webhook-site
  providerRef: webhook-site-provider
  topic: demo
- name: passthrough-tekton
  providerRef: tekton-provider
  topic: demo
```

It shows:
- Three different message providers:
  - A NATs provider
  - two REST provider
    - The first provider is for webhook.site, a website that displays your webhook events.
    - The second provider is for a Tekton event listener
- Three different event destinations:
  - A destination named "github" that uses the NATS provider to send messages on a NATs topic called "github".
  - a destination called passthrough-webhook-site that uses the webhook-site-provider to send messages to its REST endpoint.
  - a destination called passthrough-tekton that uses the REST tekton-provider to send messages to a Tekton event listener.


### Webhook Component

#### Wbhook Configuration

A webhook listener is not created by default unless it is enabled in the custom resource definition (CRD) used to create a Kabanero instance.  Below is a sample CRD that creates a new Kabanero instance that also enables the webhook component.

```
apiVersion: kabanero.io/v1alpha1
kind: Kabanero
metadata:
  name: kabanero
  namespace: kabanero
spec:
  collections:
    repositories:
      - activateDefaultCollections: true
        name: incubator
        url: https://github.com/kabanero-io/collections/releases/download/0.4.0/kabanero-index.yaml
  webhook:
    enable: true
  version: 0.4.0
```

If the webhook component is enabled, a route is also created. You can get more information about the route via `oc get route kabanero-events -o yaml`.

The webhook component expects to receive POST requests in JSON format, and creates a new JSON event that contains:
```
"webhook": { <JSON body that was received>},
"header": { <header that was received>}
```

Currently all events received by the webhook component are sent to the destination `github` defined in `eventDefinitions.yaml`.


#### Github Webhook

Github webhooks may be configured at either a per-repository level, or at an organization level. To configure at a per-repository level, follow these instructions: https://developer.github.com/webhooks/creating/#setting-up-a-webhook

To configure an organizational webhook, follow these instructions: https://help.github.com/en/github/setting-up-and-managing-your-enterprise-account/configuring-webhooks-for-organization-events-in-your-enterprise-account

Note these configurations:
- Use the hostname of the exported route for kabanero-events as the hostname for the URL of the webhook. For example, `https://kabaner-webhook-kabanero.myhost.com`
- Use "aplication/JSON" as the content type.
- The Kabanero webhook listener does not currently verify the secret you configure in Github.
- The default configuration uses Openshift auto-generated service serving self-signed certificate. Unless you had replaced the route with a certificate signed by a public certificate authority, when configuring webhook you need to choose the `disable SSL` option to skip certificate verification.

### The Events Component

The Events component provides message mediation, transformation, and actions based on incoming events.  It uses `Common Express Language` (https://opensource.google/projects/cel) as the underlying framework to
- Filter events
- Initial actions based on events
- Define new events as JSON data structure.


#### Configuring Events Framework

The events component is configured via additional files in the Kabanero collection. The file `kabanero-index.yaml` in the collection contains a pointer to the directory of those files. For example,

```
stacks:
...
triggers:
 - description: triggers for this collection
   url: https://raw.githubusercontent.com/kabanero-io/release/<release>/triggers.tar.gz
```


All the yaml files in the directory are read and processed for event trigger definitions.

#### Event Trigger Definitions

An event trigger definition file contains one or more of these sections:
- The settings section to set options.
- An event trigger section to define how to process events
- A function section for user defined functions.

##### Settings Section

The setting section supports the following options:
- dryrun: if true, will not execute actions.

For example:
```
settings:
  dryrun: false
```

##### event Triggers section

The event triggers section specifies how events are to be processed. The syntax is:

```
eventTriggers:
  - eventSource: <name of destination>
    input: <variable name of input JSON message>
    body:
     <statements>
```

For example, the following section is used to process messages from the event destination "github".  The input JSON message is stored in the variable `message` before being processed by the statements in the body.
```
- eventSource: github
  input: message
  body:
    <statements>
```

##### Function section

The function section defines a new user defined function.
```
functions:
  - name: <name of cuntion>
    input: <input variable>
    output: <outoput variable>
    body:
      <statements>
```

For example, the following snippet declares a new function with name `processGithubWebhook`, input variable `message`, and output variable `build`.

```
functions:
  - name: preprocessGithubWebhook
    input: message
    output: build
    body:
      ...
```

##### Statements

The following statements are supported:
- assignment
- if
- switch

###### Assignment Statement

An assignment statement looks like:
```
- <variable>: ' <expression>'
```

For example:
```
  - build.pr.allowedBranches: ' [ "master" ] '
  - build.push.allowedBranches : ' [ "master" ] '
  - build.tag.pattern : '"\\d\\.\\d\\.\\d"'
```

The above example shows that:
- Variables may be nested, and the result is a JSON data structure. For example, `build.pr.allowedBranches` creates a data structure where the name of the variable is `build`, and the content is a JSON data structure:
```
{
  "pr": {
      "allowedBranches": [ "master"]
  }
}
```
- The value of the variable is a CEL expression.  
- Escape characters are required for a string when parsing via CEL. For example, "\\d\\.\\d\\.\\d", after parsing, becomes "\d.\d.\d", a RegEx2 pattern for digits separated by ".".


###### if Statement

An if statement looks like:
```
- if : <condition>
  <assignment>
```

or

```
- if : <condition>
  body:
     - <statement>
```
The condition is any CEL expression that evaluates to a boolean.


###### Switch Statement

A switch statement looks like:

```
- switch:
  - <if statement>
  - <if statement>
  ...
  - default:
    body:
      - <statement>
```

Each if statement is evaluated in order.  When the first if statement whose conditional expression evaluates to true is found, its body is evaluated, and the evaluattion of the statement is complete. The body of the default is evaluated only when no other conditional expression for the if statements evaluates to true.
    

##### Build-in functions

###### filter

The filter function returns a new map or array with some elements of the original map or array filtered out.

Input:
- message: a map or array data structure
- conditional: CEL expression to evaluate each element of the data structure. If it evaluates to true, the element is kept in the returned data structure. Otherwise, it is discarded. For a map, the variable `key` is bound to the key of the element being evaluated, and the `value` variable is bound to the value. For an array, only the `value` variable is available.

Output: 
- A copy of the original data structure with some elements filtered out based on the condition.

Examples:

This example keeps only those elements of the input `header` variable that is set by github:
```
 - newHader : ' filter(header, " key.startsWith(\"X-Github\") || key.startsWith(\"github\")) '
 ```


 This example keeps only those elements of an integer array whose value is less than 10:
```
   - newArray: ' filter(oldArray, " value < 10 " )
```

###### call
The call function is used to call a user defined function.

input:
- name: name of the function
- param: parameter for the function

output:
- return value from the function


Example:

The function `sum` implements a recursive function to calculate sum of all numbers from 1 to input:
```
functions:
  - name: sum
    input: input
    output: output
    body:
      - switch:
          - if : 'input <= 0'
            output : input
          - default:
            - output: ' input + call("sum", input- 1)'
```


###### sendEvent

The sendEvent function sends an event to a destination.

Input:
  - destination: destination to send the event
  - message: a JSON compatible message
  - context : optional context for the event, such as http header

Output: empty string if OK, otherwise, error message

Example:
```
  - result: " sendEvent("tekton-listener", message,  header)
```

###### applyResources

The applyResources function applys go template substitution to resource manifests in a directory and then applies them to Kubernetes.

Input:
  - dir: directory containing the go templates
  - variable : variable for go template substitution

Return:
  Return: empty string if OK, otherwise, error message


###### kabaneroConfig

The kabaneroConfig function returns configuration for current instance of Kabanero.

Output: a map containing the Kabanero configuration.

Currently, the output contains the following:
- namespace: namespace where Kabanero instance is running 

###### jobID

The jobID function returns a new unique string each time it is called.


###### downloadYAML

The downloadYML function is used to download a YAML file from a github repository.

Input:
   - webhookMessage: original webhook message from github as sent by the Kabanero webhook component.
   - fileNameVal: name of file to download

Output: A map with the following keys:
   - error: if set, the error message encountered
   - exists: true if file exists, assuming no error
   - content: actual content of the file

Example:
The following example downloads a file named .appsody-config.yaml, and only proceeds if there were no errors and the file exists:

```
- result: "downloadYAML(message, '.appsody-config.yaml')"
- if : " ! has(result.error) "
  body:
    - if : " result.exists" 
      - body:
        ...
```


###### toDomainName

The toDomainName function converts a string into domain name format.

Input: a string
Output: the string converted to domain name format 

###### toLabel


The toLabel function converts a string in to Kubernetes label format.

Input: a string
Output: the string converted to label format 

###### split

Split a string into an array of string

Input: 
  - str: string to split
  - separator: the separator to split
Output: array of string containing original string separated by the separator.

Example:
```
  - components: " split('a/b/c', '/') "
```

After split, the variable components contains [ "a", "b", "c" ]


<a name="Building"></a>
## Building

There are two ways to build the code:
- Building in a docker container
- locally on your laptop or desktop

### Docker build

To build in a docker container:
- Clone this repository
- Install version of docker that supports multi-stage build.
- Run `./build.sh` to produces an image called `kabanero-events`.  This image is to be run in an openshift environment. An official build pushes the image as kabanero/kabanero-events and it is installed as part of Kabanero operator.

### Local build

#### Set up the build environment
To set up a local build environment:
- Install `go`
- Install `dep` tool
- Install `golint` tool
- Clone this repository into $GOPATH/src/github.com/kabanero-events
- Run `dep ensure --vendor-only` to generate the prerequisite vendor files.

#### Local development and unit test

##### Building Locally

Run `go test` to run unit test

Run `go build` to build the executable `kabanero-events`.

If you import new prerequisites in your source code:
- run `dep ensure` to regenerate the vendor directory, and `Gopkg.lock`, `Gopkg.toml`.  
- Re-run both the unit test and buld.
- Run `golint` to ensure it's lint free.
- Push the updated `Gopkg.lock` and `Gopkg.toml` if any. 

##### Testing with an Existing Kabanero Collection

To test locally outside of of a pod with existing event triggers in a collection:
- Install and configure Kabanero foundation: `https://kabanero.io/docs/ref/general/installing-kabanero-foundation.html`. Also go through the optional section to make sure you can trigger a Tekton pipeline .
- Ensure you have kubectl configured and you are able to connect to an Openshift API Server.
- `kabanero-events -master <path to openshift API server> -v <n>`,  where the -v option is the client-go logging verbosity.
- To test webhook, create a new webhook to point to your local machine's host and port. For example, `https://my-host:9080/webhook`

##### Testing with Event Triggers in a sandbox

The subdirectories under the directory `test_data/sandbox` contains sandboxes. For example, `test_data/sandbox/sample1` is a sandbox. 

To set up your sandbox: 
- Create your own sandbox repository.
-  Copy one of the sample sandboxes into your repository, say `sample1`.
- Modify or create one or more subdirectories under `sample1`, each containing Kubernetes resources to be applied when an event trigger fires.
- Create your `sample1.tar.gz` file: change directory to `sample1/triggers` and run the command `tar -cvzf ../sample1.tar.gz *`.  Push the changes.
- Edit kabanero-index.yaml and modify the url under the triggers section to point to your URL of your sample1.tar.gz. Push the changes to your reposiotyr. For example:
```
triggers:
 - description: triggers for this collection
   url: https://raw.githubusercontent.com/<owner>/kabanero-events/<barnch>/master/sample1/sample1.tar.gz
```

To set up the kabanero webhook to use the sandbox:
- From the browser, browse to kabanero-index.yaml file.
- Click on `raw` button and copy the URL in the browser. 
- Export a new environment variable: `export KABANERO_INDEX_URL=<url>`. For example, `export KABANERO_INDEX_URL=https://raw.githubusercontent.com/<owner>/<repo>/master/sample1/kabanero-index.yaml`

To run the kabanero-events in a sandbox:
- Ensure the non-sandbox version is working.
- Ensure you can run `kubectl` against your Kubernetes API server.
- Run `kabanero-events -disableTLS -master <API server URL> -v <n>`, where n is the Kubernetes log level.
- create a new webhook that points to the URL of your sandbox build.

To update your sandbox event triggers:
- Make changes to the files under the  sandbox `triggers` subdirectory
- Re-create `sample1.tar.gz`
- Push the changes
- Restart kabanero-events


#### Running in OpenShift

Running a temporary copy of Kabanero Webhook in OpenShift can be done using `oc new-app` like so:
```bash
oc new-app kabanero/webhook -disableTLS -e KABANERO_INDEX_URL=<url> 
```
