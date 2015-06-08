---
title: Auto Scaling with OpenStack Heat
---

Recently, I’ve been playing around with one of OpenStacks newer components, [Heat](https://wiki.openstack.org/wiki/Heat).

For those unfamiliar, Heat is OpenStack’s orchestration service. It provides control of all OpenStack’s components (Compute, Network, Storage, Object) using it’s own domain specific language (HOT), allowing automated provisioning and lifecycle management of entire cloud service architectures.

I’m a big fan of cloud orchestration and Heat in particular.

1. It remove’s a significant barrier to cloud entry by no longer requiring an understand of the various OpenStack API endpoints and their nuances.
2. Entire cloud architecture can be easily version controlled, as everything is a file.
3. Environments become easily reproducible, allowing them to be built and destroyed instantly.

For a more detailed run down of Heat’s terminology and architecture, take a look at Scott Lowe’s post, [An Introduction to OpenStack Heat](http://blog.scottlowe.org/2014/05/01/an-introduction-to-openstack-heat/).

Arguably one of the best features of OpenStack's orchestration, is the ability to create infrastructure that can automatically scale up and down based on a set of pre-defined alarms - this feature is called AutoScalingGroups (ASG).

To demonstrate the power of ASG, I’ve put together a Heat template which demonstrates the types of autoscaling service architectures you can build with it.

To run the demo, you’ll need to have access to either an [OpenStack Public Cloud](http://www.datacentred.co.uk) or a local sandboxed OpenStack installation using [DevStack](http://docs.openstack.org/developer/devstack/). You’ll also need the OpenStack CLI tools installed, there’s a great post on [Installing OpenStack’s CLI tools on OS X](http://dischord.org/2015/03/23/installing-openstack-s-cli-tools-on-os-x) over at [dischord.org](http://dischord.org).

Once installed, fetch the HOT template from the following gist:

{% gist bilco105/86c9f97e9e06b8122d1f %}

Then instruct Heat to launch the stack:

{% highlight bash %}
$ heat stack-create -f asg_demo.yaml \
    -P key_name='Rob' \
    -P flavor='dc1.1x1' \
    -P image='Ubuntu 14.04' \
    -P public_net_id='6751cb30-0aef-4d7e-94c3-ee2a09e705eb' asg_demo
{% endhighlight %}

Here we’re passing Heat the path to our template `asg_demo.yaml`, then setting a number of parameters that the template requires. Finally we give the stack a friendly display name, `asg_demo`.

After kicking off the above command, you’ll be given a UUID and the stack will initially go into a `CREATE_IN_PROGRESS` state. You can check the state using `heat stack-list` and `heat stack-show asg_demo`. Within a few seconds, it should change to `CREATE_COMPLETE` signalling that the stack build has completed successfully - yay!

So, what has Heat just done for us? Well;

1. It’s created a new private network, creatively named ‘private_net’, with a private subnet of 192.168.10.0/24 and DHCP enabled.
2. It’s created a new router, you guessed it, called `router` and attached it to `private_net` and the external Internet network specified in the above CLI call. This provides instances on `private_net` with access to the Internet.

Now this is where the fun starts…!

1. It’s created a new AutoScalingGroup, which has the configuration of an instance (Virtual machine) associated. By default, we’ve specified we want to spawn 1 of these instances initially, and capped the maximum number of that instance it can spawn to 10.
2. It’s defined two Heat Scaling Policies, `scale_up_policy` and `scale_down_policy`. Policies are essentially unique API endpoints with an associated action. In our case, they increase or decrease the instance count by 1 respectively. Each policy generates a unique URL, which will perform the action whenever it receives a HTTP POST request. The `cooldown` parameter limits how frequently you can call each policy (i.e. once every 60 seconds).
3. Finally, it’s defined two Ceilometer alarms. These monitor the combined average CPU utilisation of all of the instances in the AutoScalingGroup. We’ve configured it so if the CPU utilisation is above 50% for 1 minute, it will call the `scale_up_policy` to spawn another instance to try and cope with demand. Alternatively, if the CPU utilisation is below 15% for 10 minutes, it will start terminating instances until either the alarm clears, or it hits the ASG’s min_size definition of 1.

Pretty cool huh?, it gets better.

The Instance template that’s part of the ASG has some cloud-init code attached. This generates fake CPU utilisation by installing and running [stress](http://people.seas.harvard.edu/~apw/stress/) for 15-minutes when the instance is first booted (see `cloud_config_stress` within the Heat template for how cloud-init is configured), and Scale Horizontally’s post for an [Introduction to cloud-init](http://www.scalehorizontally.com/2013/02/24/introduction-to-cloud-init/).

By running `nova list` you’ll see we currently only have a single instance running, as per our ASG’s min_size definition. However, keep running it `watch -n60 nova list` and you’ll eventually see new instances start to spawn as the CPU alarm is breached.

This will continue until we finally hit our max_size of 10. 15 minutes after this, the final stress command will complete. Allow another 10-15 minutes for the average CPU to drop below 15%, at which point Heat will automatically start terminating instances as they’re no longer required. We’ll eventually up with a single instance again.

Finally, you can call the Scale Up and Scale Down policies manually. To fetch the URLs, execute `heat output-list asg_demo` to list the available outputs defined at the bottom of our template, followed by `heat output-show asg_demo scale_up_url` and `heat output-show asg_demo scale_dn_url` for the respective policies.

Once you have the URLs, generate an HTTP POST request using `curl -XPOST <url>`.

I hope this post has highlighted some of the key benefits of OpenStack service orchestration. It’s a truly remarkable project, and in my opinion sets OpenStack apart from other cloud management and operating systems.

For another example of using Heat templates, take a look at [Orchestrating CoreOS with OpenStack Heat](http://dischord.org/2015/04/18/orchestrating-coreos-with-openstack-heat/).
