# S2I Workshop

## Development environment

This assumes you have the OpenShift CLI installed on your machine. It would be best if you're running version 3.6.0 (`oc version`). If you already don't have a local OpenShift cluster running then `oc cluster up` and `oc login -u system:admin`. If you can see the single node of your cluster by running `oc get nodes` then you're good to go.

## Example ruby app

We'll be using this Ruby application for our deployments https://github.com/openshift-quickstart/sinatra-example.

Optionally you can fork this repo. This will allow you to push some changes to the repo in the last step of this lab.

## The Ruby S2I builder image

You should have a Ruby S2I builder image available in the _openshift_ namespace. We'll be using this for building our example application. You can take a look the source of this image here https://github.com/sclorg/s2i-ruby-container.

We use the OpenShift stock Ruby S2I builder image.

    $ oc get is ruby -n openshift
    $ oc describe is/ruby -n openshift

## Deploy your app

Config for this deployment is split into yaml files. You should definitely take a look at each of these files or none of this will make any sense.
Components of this deployment are split into separate yaml files. We'll be using

If you don't feel at home with the various command needed for inspecting the state of your OpenShift environment (`oc get ...`, `oc describe ...` etc.) then use the web interface at https://localhost:8443 to get a better view of what's happening.

Create a new OpenShift project/namespace.

    $ oc new-project s2i-workshop

Create the image stream for the stock Ruby S2I builder image we looked at earlier.

    $ oc create -f ruby-22-centos7-is.yaml
    $ oc get is ruby-22-centos7
    $ oc describe is ruby-22-centos7

And create a second image stream for our application image.

    $ oc create -f sinatra-example-is.yaml
    $ oc get is sinatra-example
    $ oc describe is sinatra-example

Open a second terminal and run `oc get pods --watch`. Keeping an eye on this should give you a better idea of what's going on if builds get kicked off etc.

## Build config

Now we are are ready to create the build config for our application. (If you forked the repo then you need to edit the `sinatra-example-bc.yaml` file first. Update the `source.git.uri` field).

    $ oc create -f sinatra-example-bc.yaml

If this is successful then you should see the builder container start. If the build completes successfully then you should see the freshly minted application image mapped to the image stream you just created.

    $ oc describe is sinatra-example

If this failed for some reason then the first thing you should do is check the logs for this build. (`oc get builds` to get a complete list if you have already started more than one).

    $ oc describe build sinatra-example-1
    $ oc logs --version=1 bc/sinatra-example

Or you can restart the build and tail it's logs.

    $ oc start-build sinatra-example --follow

## Deployment config & service



    $ oc create -f sinatra-example-dc.yaml
    $ oc get dc sinatra-example

You should see a deploy job container launching. Once this completes then you should have a single application pod running.

    $ oc get pods

Forward a local port to the running pod and use `curl` to check that everything works as expected.

    $ oc port-forward sinatra-example-foo 8080
    $ curl -i localhost:8080

## Trigger a build via webhook

As a final step we simulate a webhook delivery. Normally this would be something that happens automatically when you push to a git repo.

If you forked the application repo then you can optionally push some change to the

Take a look at the deployment config. You should see there two webhook URLs. You need the generic URL in the next step.

    $ oc describe bc/sinatra-example

And run curl to call this webhook.

    $ curl -i -k -XPOST -H "Content-Type: application/json" https://127.0.0.1:8443/...

Now you should see that a build is triggered, and once that completes a deploy as well.

You can use port forwarding and curl as before to check that the updated version of the app is actually deployed now.

## Clean up

    $ oc delete project s2i-workshop
    $ oc cluster down
