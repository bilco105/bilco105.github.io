---
title: Automatically disable unconfigured interfaces in Junos
---

It's always a good idea to disable any unused ports on your network. But, writing the configuration out by hand is time consuming (especially on high-density devices), and quite frankly boring.

Luckily, Juniper provide an easy way for us to automate mundane tasks such as this with the power of [SLAX](http://www.juniper.net/documentation/en_US/junos14.2/topics/concept/junos-script-automation-slax-overview.html). Here's a quick script I put together which does just that.

{% gist bilco105/eeb3fa5a0f3079242164 %}

This script disables all unconfigured interfaces beginning with either ```ge``` (Gigabit Ethernet) or ```xe``` (10-Gigabit Ethernet) by applying a group creatively named ```DISABLE_INTERFACE``` to the interface.

To get this working on your own Juniper box is pretty straight forward;

  1. SCP the above gist to ```/var/db/scripts/commit```

  2. Create an apply group which disables the interface:  ```set groups DISABLE_INTERFACE interfaces <*> disable```

  3. Set the script to run at commit time  : ```set system scripts commit file disable_unconfigured_interfaces.slax```

Be aware that when you commit the above changes, the script will be run straight away and unconfigured interfaces will be disabled!

For the cautious, use ```commit check``` first, which will print out the interfaces it's going to disable. You can also double check the final configuration using ```show interfaces``` before hitting ```commit```.
