# Openstack with midonet and docker

Provides a Vagrantfile that can be used with the docker backend to run an
allinone of Openstack that uses the following plugins:
* Neutron: Midonet.
* Nova: nova-docker with the midonet vif driver.

# Usage

```
vagrant up
```

After the provisioning is done (go grab a coffee or two). You can access
[horizon](http://192.168.124.185). Username admin and the password you can
find by doing:

```
vagrant ssh -c "cat ~/keystonerc_admin" 2> /dev/null | grep OS_PASSWORD | cut
-d'=' -f2
```

It is ready to start containers with cirros and there is a default private
network called foo.


# Resources

To learn more about midonet, you can head over to
* [midonet](http://midonet.org/)

* join us at https://slack.midonet.org

* write to http://lists.midonet.org/listinfo/midonet-user

To learn more about using nova with docker (pulling docker images into glance
and such):

[Nova-docker](https://github.com/stackforge/nova-docker)
