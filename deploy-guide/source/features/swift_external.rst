Use an external Swift Proxy with the Overcloud
===============================================

|project| supports use of an external Swift proxy already available to the
operator, that may need to configure at deploy time.

This happens by enabling a particular environment file when deploying the
Overcloud, specifically `environments/swift-external.yaml`.

In the environment file above user must adjust the parameters to fit
its setup by creating a custom environment file (i.e.
*~/my-swift-settings.yaml*)::

  parameter_defaults:
     ExternalPublicUrl: 'http://<Public Swift endpoint or loadbalancer>:9024/v1/AUTH_%(tenant_id)s'
     ExternalInternalUrl: 'http://<Internal Swift endpoint>:9024/v1/AUTH_%(tenant_id)s'
     ExternalAdminUrl: 'http://<Admin Swift endpoint>:9024'
     ExternalSwiftUserTenant: 'service'
     SwiftPassword: 'choose_a_random_password'

The user can create an environment file with the required settings
and add the files above to the deploy commandline::

  openstack overcloud deploy --templates -e /usr/share/openstack-tripleo-heat-templates/environments/swift-external.yaml -e ~/my-swift-settings.yaml

Once the deploy has succeeded, user has to complete the
configuration on the external swift proxy, configuring it to use the
keystone authentication provider. This environment files creates also
a service user called *swift* that can be used for this purpose. The
password for this user is defined by using the *SwiftPassword*
parameter, as shown above.

The external Swift proxy must use Keystone from the overcloud, otherwise
authentication will fail. The public Keystone endpoint must be
accessible from the proxy therefore.

The following snippet from `/etc/swift/proxy-server.conf` is an example
how to configure the Swift proxy to use Keystone from the overcloud::

  [pipeline:main]
  pipeline = [... other middlewares ...]  authtoken keystone [... other middlewares ...]

  [filter:keystone]
  use = egg:swift#keystoneauth
  operator_roles = admin, SwiftOperator
  cache = swift.cache

  [filter:authtoken]
  paste.filter_factory = keystonemiddleware.auth_token:filter_factory
  signing_dir = /tmp/keystone-signing-swift
  www_authenticate_uri = http://<public Keystone endpoint>:5000/
  auth_url = http://<admin Keystone endpoint>:5000/
  password = <Password as defined in the environment parameters>
  auth_plugin = password
  project_domain_id = default
  user_domain_id = default
  project_name = service
  username = swift
  cache = swift.cache
  include_service_catalog = False
  delay_auth_decision = True
