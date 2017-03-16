[![Build Status](https://travis-ci.org/IBM/openwhisk-data-processing-cloudant.svg?branch=master)](https://travis-ci.org/IBM/openwhisk-data-processing-cloudant)

# OpenWhisk Hands On - Cloudant Data Processing
Learn how to [create Cloudant data processing apps](https://github.com/IBM/openwhisk-data-processing-cloudant/wiki) with Apache OpenWhisk on IBM Bluemix. This tutorial will take about 10 minutes to complete.

You should have a basic understanding of the OpenWhisk programming model. If not, [try the action, trigger, and rule demo first](https://github.com/IBM/openwhisk-action-trigger-rule). You'll also need a Bluemix account and the latest [OpenWhisk command line tool (`wsk`) installed and on your PATH](docs/OPENWHISK.md).

When complete, move on to more complex serverless applications, such as those named _OpenWhisk 201_ or tagged as [_openwhisk-use-cases_](https://github.com/search?q=topic%3Aopenwhisk-use-cases+org%3AIBM&type=Repositories).

# Cloudant data processing with OpenWhisk
The example shows how to write an action that inserts data into Cloudant and how to create another action to respond to that database change.

It also shows how to use built-in actions, such as those provided by the `/whisk.system/cloudant` package along with custom actions in a _sequence_ to link units of logic in a series.

![High level diagram](docs/data-processing-cloudant.png)

Steps

1. [Provision Cloudant](#1-provision-cloudant)
2. [Create OpenWhisk actions, triggers, and rules](#2-create-openwhisk-actions-triggers-and-rules)
3. [Test database change events](#3-test-database-change-events)
4. [Delete actions, triggers, and rules](#4-delete-actions-triggers-and-rules)
5. [Recreate deployment manually](#5-recreate-deployment-manually)

# 1. Provision Cloudant
Log into Bluemix, provision a Cloudant database instance, and name it `openwhisk-cloudant`. Log into the Cloudant web console and create a database named `cats`.

Copy `template.local.env` to a new file named `local.env` and update the `CLOUDANT_INSTANCE`, `CLOUDANT_DATABASE`, `CLOUDANT_USERNAME`, and `CLOUDANT_PASSWORD` for your instance.

# 2. Create OpenWhisk actions, triggers, and rules
`deploy.sh` is a convenience script reads the environment variables from `local.env` and creates the OpenWhisk actions, triggers, and rules on your behalf. Later you will run these commands yourself.

```bash
./deploy.sh --install
```
> **Note**: If you see any error messages, refer to the [Troubleshooting](#troubleshooting) section below.

> **Note**: `deploy.sh` will be replaced with [`wskdeploy`](https://github.com/openwhisk/openwhisk-wskdeploy) in the future. `wskdeploy` uses a manifest to deploy declared triggers, actions, and rules to OpenWhisk.

# 3. Test database change events
To test, invoke the first action manually. Open one terminal window to poll the logs:
```bash
wsk activation poll
```

And in a second terminal, invoke the action:
```bash
wsk action invoke --blocking --result write-to-cloudant
```

# 4. Delete actions, triggers, and rules
Use `deploy.sh` again to tear down the OpenWhisk actions, triggers, and rules. You will recreate them step-by-step in the next section.

```bash
./deploy.sh --uninstall
```

# 5. Recreate deployment manually
This section provides a deeper look into what the `deploy.sh` script executes so that you understand how to work with OpenWhisk triggers, actions, rules, and packages in more detail.

## 5.1 Bind Cloudant package with credential parameters
Make the Cloudant instance in Bluemix available as an event source.

```bash
wsk package bind /whisk.system/cloudant "$CLOUDANT_INSTANCE" \
  --param username "$CLOUDANT_USERNAME" \
  --param password "$CLOUDANT_PASSWORD" \
  --param host "$CLOUDANT_USERNAME.cloudant.com"
```

## 5.2 Create trigger to fire events when data is inserted
Create a trigger named `image-uploaded` that subscribes to that Cloudant instance and specific database change events.

```bash
wsk trigger create image-uploaded \
  --feed "/_/$CLOUDANT_INSTANCE/changes" \
  --param dbname "$CLOUDANT_DATABASE"
```

## 5.3 Create action that is manually invoked to write to database
Upload the action code named `write-to-cloudant` that inserts text and an image to Cloudant which will initiate a database change event.

```bash
wsk action create write-to-cloudant actions/write-to-cloudant.js \
  --param CLOUDANT_USERNAME "$CLOUDANT_USERNAME" \
  --param CLOUDANT_PASSWORD "$CLOUDANT_PASSWORD" \
  --param CLOUDANT_DATABASE "$CLOUDANT_DATABASE"
```

## 5.4 Create action to respond to database insertions
Upload the action code named `write-from-cloudant` that will receive database change information and log it to the console.

```bash
wsk action create write-from-cloudant actions/write-from-cloudant.js
```

## 5.5 Create sequence that ties database read to handling action
Specify a linkage between the built-in Cloudant `read` action and the custom `write-from-cloudant` above in a sequence named `write-from-cloudant-sequence`.

```bash
wsk action create write-from-cloudant-sequence \
  --sequence /_/$CLOUDANT_INSTANCE/read,write-from-cloudant
```

## 5.6 Create rule that maps database change trigger to sequence
Declare a rule named `echo-images` that maps the trigger `image-uploaded` to the action sequence `write-from-cloudant-sequence`. Without this mapping, the trigger will fire but no logic will be executed in response.

```bash
wsk rule create echo-images image-uploaded write-from-cloudant-sequence
```

# Troubleshooting
Check for errors first in the OpenWhisk activation log. Tail the log on the command line with `wsk activation poll` or drill into details visually with the [monitoring console on Bluemix](https://console.ng.bluemix.net/openwhisk/dashboard).

If the error is not immediately obvious, make sure you have the [latest version of the `wsk` CLI installed](https://console.ng.bluemix.net/openwhisk/learn/cli). If it's older than a few weeks, download an update.
```bash
wsk property get --cliversion
```

# License
[Apache 2.0](LICENSE.txt)
