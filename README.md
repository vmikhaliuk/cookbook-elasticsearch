# Elasticsearch Chef Cookbook

[![Build Status](https://travis-ci.org/elastic/cookbook-elasticsearch.svg?branch=master)](https://travis-ci.org/elastic/cookbook-elasticsearch) [![Cookbook Version](https://img.shields.io/cookbook/v/elasticsearch.svg)](https://supermarket.chef.io/cookbooks/elasticsearch)[![Build Status](https://jenkins-01.eastus.cloudapp.azure.com/job/elasticsearch-cookbook/badge/icon)](https://jenkins-01.eastus.cloudapp.azure.com/job/elasticsearch-cookbook/)

**Please** review the [frequently asked questions](FAQ.md) and [contributing guidelines](CONTRIBUTING.md) before opening issues or submitting pull requests.

## Looking for Elasticsearch 6.0.0?

Please [try the 4.0.0-beta release](https://github.com/elastic/cookbook-elasticsearch/tree/4.0.0-beta) of this cookbook. Otherwise pin your cookbook back to a '~> 3.0' release. The 4.0.0-beta branch will be promoted to master once it's stable & working with Elasticsearch 6.0 releases.

## Attributes

Please consult [attributes/default.rb](attributes/default.rb) for a large list
of checksums for many different archives and package files of different
elasticsearch versions. Both recipes and resources/providers here use those
default values.

You may use `%s` in your URL and this cookbook will use sprintf/format to insert
the version parameter as a string into your download_url.

|Name|Default|Other values|
|----|-------|------------|
|`default['elasticsearch']['download_urls']['debian']`|[See values](attributes/default.rb).|`%s` will be replaced with the version attribute above|
|`default['elasticsearch']['download_urls']['rhel']`|[See values](attributes/default.rb).|`%s` will be replaced with the version attribute above|
|`default['elasticsearch']['download_urls']['tarball']`|[See values](attributes/default.rb).|`%s` will be replaced with the version attribute above|

This cookbook's `elasticsearch::default` recipe also supports setting any `elasticsearch_` resource using attributes:

```
default['elasticsearch']['user'] = {}
default['elasticsearch']['install'] = {}
default['elasticsearch']['configure'] = {}
default['elasticsearch']['service'] = {}
default['elasticsearch']['plugin'] = {}
```

For example, this will pass a username 'foo' to `elasticsearch_user` and set a uid to `1234`:
```
default['elasticsearch']['user']['username'] = 'foo'
default['elasticsearch']['user']['uid'] = '1234'
```

## Recipes

Resources are the intended way to consume this cookbook, however we have
provided a single recipe that configures Elasticsearch by downloading an archive
containing a distribution of Elasticsearch, and extracting that into `/usr/share`.

See the attributes section above to for what defaults you can adjust.

### default

The default recipe creates an elasticsearch user, group, package installation,
configuration files, and service with all of the default options.

Please note that there are [additional examples within the test fixtures](test/fixtures/cookbooks/elasticsearch_test),
including a demonstration of how to configure two instances of Elasticsearch on a single server.

## Resources

## Notifications and Service Start/Restart

The resources provided in this cookbook **do not automatically restart** services when changes have occurred. They ***do start services by default when configuring a new service*** This has been done to protect you from accidental data loss and service outages, as nodes might restart simultaneously or may not restart at all when bad configuration values are supplied.

elasticsearch_service has a special `service_actions` parameter you can use to specify what state the underlying service should be in on each chef run (defaults to `:enabled` and `:started`). It will also pass through all of the standard `service` resource
actions to the underlying service resource if you wish to notify it.

You **must** supply your desired notifications when using each resource if you want Chef to automatically restart services. Again, we don't recommend this unless you know what you're doing.

We are supporting whyrun mode in this cookbook, simply because we're using all builtin resources from core Chef, and these also already support whyrun. If you contribute to this cookbook, please be sure to maintain that or guard dangerous Ruby code with something like `if !whyrun_mode? || nested_resource.whyrun_supported?`.

### Resource names

Many of the resources provided in this cookbook need to share configuration
values. For example, the `elasticsearch_service` resource needs to know the path
to the configuration file(s) generated by `elasticsearch_configure` and the path
to the actual ES binary installed by `elasticsearch_install`. And they both need
to know the appropriate system user and group defined by `elasticsearch_user`.

Search order: In order to make this easy, all resources in this cookbook use the following
search order to locate resources that apply to the same overall
Elasticsearch setup:

1. Resources that share the same resource name
1. Resources that share the same value for `instance_name`
1. Resources named `default` or resources named `elasticsearch`
   - This fails if both `default` and `elasticsearch` resources exist

Examples of more complicated resource names are left to the reader, but here we
present a typical example that should work in most cases:

```ruby
elasticsearch_user 'elasticsearch'
elasticsearch_install 'elasticsearch'
elasticsearch_configure 'elasticsearch'
elasticsearch_service 'elasticsearch'
elasticsearch_plugin 'x-pack'
```

### elasticsearch_user
Actions: `:create`, `:remove`

Creates a user and group on the system for use by elasticsearch. Here is an
example with many of the default options and default values (all options except
a resource name may be omitted).

Examples:

```ruby
elasticsearch_user 'elasticsearch'
```

```ruby
elasticsearch_user 'elasticsearch' do
  username 'elasticsearch'
  groupname 'elasticsearch'
  shell '/bin/bash'
  comment 'Elasticsearch User'

  action :create
end
```

### elasticsearch_install
Actions: `:install`, `:remove`

Downloads the elasticsearch software, and unpacks it on the system. There are
currently three ways to install -- `'repository'` (the default), which creates an
apt or yum repo and installs from there, `'package'`, which downloads the appropriate
package from elasticsearch.org and uses the package manager to install it, and
`'tarball'` which downloads a tarball from elasticsearch.org and unpacks it.
This resource also comes with a `:remove` action which will remove the package
or directory elasticsearch was unpacked into.

You may always specify a download_url and/or download_checksum, and you may
include `%s` which will be replaced by the version parameter you supply.

Please be sure to consult the above attribute section as that controls how
Elasticsearch version, download URL and checksum are determined if you omit
them.

**NOTE**: The `:remove` action has not been implemented yet. Pull requests are
very much welcome & encouraged, if you'd like to see this feature.

Examples:

```ruby
elasticsearch_install 'elasticsearch'
```

```ruby
elasticsearch_install 'my_es_installation' do
  type 'package' # type of install
  version '5.6.5'
  action :install # could be :remove as well
end
```

```ruby
elasticsearch_install 'my_es_installation' do
  type 'tarball' # type of install
  dir '/usr/local' # where to install

  download_url "https://download.elasticsearch.org/elasticsearch/elasticsearch/elasticsearch-1.7.2.tar.gz"
  # sha256
  download_checksum "6f81935e270c403681e120ec4395c28b2ddc87e659ff7784608b86beb5223dd2"

  action :install # could be :remove as well
end
```

```ruby
elasticsearch_install 'my_es_installation' do
  type 'tarball' # type of install
  version '5.6.5'
  action :install # could be :remove as well
end
```

```ruby
elasticsearch_install 'my_es_installation' do
  type 'package' # type of install
  download_url "https://download.elasticsearch.org/elasticsearch/elasticsearch/elasticsearch-1.7.2.deb"
  # sha256
  download_checksum "791fb9f2131be2cf8c1f86ca35e0b912d7155a53f89c2df67467ca2105e77ec2"
  instance_name 'elasticsearch'
  action :install # could be :remove as well
end
```

### elasticsearch_configure
Actions: `:manage`, `:remove`

Configures an elasticsearch instance; creates directories for configuration,
logs, and data. Writes files `log4j2.properties`, `elasticsearch.in.sh` and
`elasticsearch.yml`.

The main attribute for this resource is `configuration`,
which is a hash of any elasticsearch configuration directives. The
other important attribute is `default_configuration` -- this contains the
minimal set of required defaults.

Note that these are both _not_ a Chef mash, everything must be in a single level
of keys and values. Any settings you pass in configuration will be merged into
(and potentially overwrite) any default settings.

See the examples, [as well as the attributes in the resource file](libraries/resource_configure.rb),
for more.

Examples:

With all defaults -
```ruby
elasticsearch_configure 'elasticsearch'
```

With mostly defaults -
```ruby
elasticsearch_configure 'elasticsearch' do
    allocated_memory '512m'
    configuration ({
      'cluster.name' => 'escluster',
      'node.name' => 'node01',
      'http.port' => 9201
    })
end
```

Very complicated -
```ruby
elasticsearch_configure 'my_elasticsearch' do
  # if you override one of these, you probably want to override all
  path_home     "/opt/elasticsearch"
  path_conf     "/etc/opt/elasticsearch"
  path_data     "/var/opt/elasticsearch"
  path_logs     "/var/log/elasticsearch"
  path_pid      "/var/run/elasticsearch"
  path_plugins  "/opt/elasticsearch/plugins"
  path_bin      "/opt/elasticsearch/bin"

  # override logging parameters
  cookbook_log4j2_properties "my_wrapper_cookbook"
  template_log4j2_properties "my_log4j2.properties.erb"

  logging({:"action" => 'INFO'})

  allocated_memory '123m'

  jvm_options %w(
                -XX:+UseParNewGC
                -XX:+UseConcMarkSweepGC
                -XX:CMSInitiatingOccupancyFraction=75
                -XX:+UseCMSInitiatingOccupancyOnly
                -XX:+HeapDumpOnOutOfMemoryError
                -XX:+PrintGCDetails
              )

  configuration ({
    'node.name' => 'crazy'
  })

  action :manage
end
```

### elasticsearch_service
Actions: `:configure`, `:remove`

Writes out a system service configuration of the appropriate type, and enables
it to start on boot. You can override almost all of the relevant settings in
such a way that you may run multiple instances. Most settings will be taken from
a matching `elasticsearch_config` resource in the collection.

```ruby
elasticsearch_service 'elasticsearch'
```

If you'd like to skip init scripts and systemd scripts, simply pass `nil` for
the template file (init_source or systemd_source) and this cookbook will
entirely skip trying to setup those scripts. Combined with changing the default
service actions, this will have the same effect as `action :nothing`.

### elasticsearch_plugin
Actions: `:install`, `:remove`

Installs or removes a plugin to a given elasticsearch instance and plugin
directory. Please note that there is currently no way to upgrade an existing
plugin using commandline tools, so we haven't exposed that feature here either.
Furthermore, there isn't a way to determine if a plugin is compatible with ES or
even what version it is. So once we install a plugin to a directory, we
generally assume that is the desired one and we don't touch it further.

See https://github.com/elastic/cookbook-elasticsearch/issues/264 for more info.
NB: You [may encounter issues on certain distros](http://blog.backslasher.net/java-ssl-crash.html) with NSS 3.16.1 and OpenJDK 7.x.

Officially supported or commercial plugins require just the plugin name:

```ruby
elasticsearch_plugin 'analysis-icu' do
  action :install
end
elasticsearch_plugin 'shield' do
  action :install
end
```

Plugins from GitHub require a URL of 'username/repository' or 'username/repository/version':

```ruby
elasticsearch_plugin 'kopf' do
  url 'lmenezes/elasticsearch-kopf'
  action :install
end

elasticsearch_plugin 'kopf' do
  url 'lmenezes/elasticsearch-kopf/1.5.7'
  action :install
end
```

Plugins from Maven Central or Sonatype require 'groupId/artifactId/version':
```ruby
elasticsearch_plugin 'mapper-attachments' do
  url 'org.elasticsearch/elasticsearch-mapper-attachments/2.6.0'
  action :install
end
```

Plugins can be installed from a custom URL or file location as follows:
```ruby
elasticsearch_plugin 'mapper-attachments' do
  url 'http://some.domain.name//my-plugin-1.0.0.zip'
  action :install
end

elasticsearch_plugin 'mapper-attachments' do
  url 'file:/path/to/my-plugin-1.0.0.zip'
  action :install
end
```

The plugin resource respects the `https_proxy` or `http_proxy` (non-SSL)
[Chef settings](https://docs.chef.io/config_rb_client.html) unless explicitly
disabled using `chef_proxy false`:
```ruby
elasticsearch_plugin 'kopf' do
  url 'lmenezes/elasticsearch-kopf'
  chef_proxy false
  action :install
end
```

To run multiple instances per machine, an explicit `plugin_dir` location
has to be provided:

```ruby
elasticsearch_plugin 'x-pack' do
  plugin_dir '/usr/share/elasticsearch_foo/plugins'
end
```

If for some reason, you want to name the resource something else, you may
provide the true plugin name using the `plugin_name` parameter:

```ruby
elasticsearch_plugin 'xyzzy' do
  plugin_name 'kopf'
  url 'lmenezes/elasticsearch-kopf'
  action :install
end
```

## License

This software is licensed under the Apache 2 license, quoted below.

    Copyright (c) 2015 Elasticsearch <https://www.elastic.co/>

    Licensed under the Apache License, Version 2.0 (the "License");
    you may not use this file except in compliance with the License.
    You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

    Unless required by applicable law or agreed to in writing, software
    distributed under the License is distributed on an "AS IS" BASIS,
    WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    See the License for the specific language governing permissions and
    limitations under the License.
