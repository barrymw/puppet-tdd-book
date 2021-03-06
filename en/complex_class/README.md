# Create a more complex class and refactor it

Now that we have all our [setup](setup/README.md) done and a [functional class](simple_class/README.md) to manage the OpenLDAP client part, let’s deal with a more complex part: the server class.

## WRITE THE ACCEPTANCE TESTS

Let’s write the acceptance tests to code the behavior we want in `spec/acceptance/openldap__server_spec.rb`:

```ruby
require 'spec_helper_acceptance'
 
describe 'openldap::server' do
  describe 'running puppet code' do
    it 'should work with no errors' do
      pp = <<-EOS
        class { 'openldap::server': }
      EOS
 
      # Run it twice and test for idempotency
      apply_manifest(pp, :catch_failures => true)
      apply_manifest(pp, :catch_changes => true)
    end
 
    describe port(389) do
      it { is_expected.to be_listening }
    end
 
    describe service('slapd') do
      it { is_expected.to be_enabled }
      it { is_expected.to be_running }
    end
  end
end
```

`openldap::server`, the server is listening on port 389 and that slapd service is running and enabled at boot time. No need to run it for now because it will obviously fail as the `openldap::server` does not exist yet.

## WRITE THE UNIT TESTS

Let’s write the unit tests skeleton for our `openldap::serverclass` in `spec/classes/openldap__server_spec.rb`:

```ruby
require 'spec_helper'
 
describe 'openldap::server' do
 
  on_supported_os.each do |os, facts|
    context "on #{os}" do
      let(:facts) do
        facts
      end
 
      it { is_expected.to compile.with_all_deps }
      it { is_expected.to contain_service('slapd')
        .that_requires('Package[openldap-servers]')
      }
 
      case facts[:osfamily]
      when 'Debian'
        it { is_expected.to contain_package('openldap-servers').with(
          {
            :ensure => :present,
            :name   => 'slapd',
          }
        ) }
      else
        it { is_expected.to contain_package('openldap-servers').with(
          {
            :ensure => :present,
            :name   => 'openldap-servers',
          }
        ) }
      end
    end
  end
end
```

When we declare the `openldap::server` class, we want the catalog to contain a service resource named `slapd` that requires a package named `slapd` on Debian and `openldap-servers` on RedHat.

## WRITE THE PUPPET CODE

Now, let’s write the actual Puppet code:

```puppet
class openldap::server {
  $package_name = $::osfamily ? {
    'Debian' => 'slapd',
    'RedHat' => 'openldap-servers',
  }
  package { 'openldap-servers':
    ensure => present,
    name   => $package_name,
  } ->
  service { 'slapd':
    ensure => running,
    enable => true,
  }
}
```

## LAUNCH THE UNIT TESTS

Now, if you launch the unit tests, it should work:

```shell
$ bundle exec rake spec SPEC_OPTS=-fd SPEC=spec/classes/openldap__server_spec.rb
...
openldap::server
  on debian-7-x86_64
    should compile the catalogue without cycles
    should contain Service[slapd]
    should contain Package[openldap-servers] with ensure => :present and name => "slapd"
  on redhat-7-x86_64
    should compile the catalogue without cycles
    should contain Service[slapd]
    should contain Package[openldap-servers] with ensure => :present and name => "openldap-servers"
...
Finished in 1.27 seconds (files took 0.67818 seconds to load)
6 examples, 0 failures
```

## LAUNCH THE ACCEPTANCE TESTS

### ON REDHAT 7

Let’s test on RedHat:

```shell
BEAKER_set=centos-7-x86_64-vagrant bundle exec rspec spec/acceptance/openldap__server_spec.rb
...
openldap::server
  running puppet code
localhost $ scp /tmp/beaker20150305-11384-7chojc centos-7-x64:/tmp/apply_manifest.pp.XqY8IJ {:ignore => }
localhost $ scp /tmp/beaker20150305-11384-1opmurv centos-7-x64:/tmp/apply_manifest.pp.hLyO5k {:ignore => }
    should work with no errors
    Port "389"
      should be listening
    Service "slapd"
      should be enabled
      should be running
Destroying vagrant boxes
==> centos-7-x64: Forcing shutdown of VM...
==> centos-7-x64: Destroying VM and associated drives...
 
Finished in 19.05 seconds (files took 2 minutes 17.4 seconds to load)
4 examples, 0 failures
```

### ON DEBIAN 7

```shell
BEAKER_set=debian-7-x86_64-vagrant bundle exec rspec spec/acceptance/openldap__server_spec.rb
...
openldap::server
  running puppet code
localhost $ scp /tmp/beaker20150305-13082-1sffkj5 debian-7-x64:/tmp/apply_manifest.pp.5AjHHb {:ignore => }
localhost $ scp /tmp/beaker20150305-13082-9tjbgg debian-7-x64:/tmp/apply_manifest.pp.1HcVFQ {:ignore => }
    should work with no errors
    Port "389"
      should be listening
    Service "slapd"
      should be enabled
      should be running
Destroying vagrant boxes
==> debian-7-x64: Forcing shutdown of VM...
==> debian-7-x64: Destroying VM and associated drives...
 
Finished in 17.89 seconds (files took 1 minute 31.4 seconds to load)
4 examples, 0 failures
```

## REFACTOR

Now that everything is working as expected, we can refactor without risking any regression.

For the moment, our class is quite simple, but it might become wildly unmaintainable as it grows. To prevent that, we’ll use the more standard `package -> config ~> service` pattern explained in R.I.Pienaar’s excellent blog post ["Simple Puppet Module Structure Redux"](https://www.devco.net/archives/2012/12/13/simple-puppet-module-structure-redux.php):

```shell
$ mkdir manifests/server
```

The `openldap::server` class becomes (we don’t manage configuration for now):

```puppet
class openldap::server {
  class { '::openldap::server::install': } ->
  class { '::openldap::server::config': } ~>
  class { '::openldap::server::service': } ->
  Class['openldap::server']
}
```

The `openldap::server::install` class:


```puppet
class openldap::server::install {
  $package_name = $::osfamily ? {
    'Debian' => 'slapd',
    'RedHat' => 'openldap-servers',
  }
  package { 'openldap-servers':
    ensure => present,
    name   => $package_name,
  }
}
```

The `openldap::server::config` class (empty for now):

```puppet
class openldap::server::config {
}
```

And finally, the `openldap::server::service` class:


```puppet
class openldap::server::service {
  service { 'slapd':
    ensure => running,
    enable => true,
  }
}
```

## LAUNCH THE UNIT TESTS

And now we can launch the unit tests to see if we didn’t break anything:
Unfortunately, as of version 2.0.1, rspec-puppet does not look up in the full graph for resource dependencies, so we can not yet validate that we don’t break resource application order when refactoring (corresponding thread in [puppet-dev mailing list](https://groups.google.com/forum/#!topic/puppet-dev/76nGtIKNkI8)).

We will mark this test as pending for now. That way, our test suite will pass even though this specific test fails, but it will fail again when rspec-puppet is fixed, so that we’re warned. In the meantime, we’ll add some tests that check the dependency between subclasses. This roughly checks the same thing but is less than ideal as the module’s structure should not appear in the tests.

```ruby
require 'spec_helper'
 
describe 'openldap::server' do
 
  on_supported_os.each do |os, facts|
    context "on #{os}" do
      let(:facts) do
        facts
      end
 
      it { is_expected.to compile.with_all_deps }
      it {
        pending 'rspec-puppet does not support recursive dependency yet.'
        is_expected.to contain_service('slapd')
          .that_requires('Package[openldap-servers]')
      }
 
      it { is_expected.to contain_service('slapd') }
      it { is_expected.to contain_class('openldap::server::install')
        .that_comes_before('Class[openldap::server::config]')
      }
      it { is_expected.to contain_class('openldap::server::config')
        .that_notifies('Class[openldap::server::service]')
      }
 
      case facts[:osfamily]
      when 'Debian'
        it { is_expected.to contain_package('openldap-servers').with(
          {
            :ensure => :present,
            :name   => 'slapd',
          }
        ) }
      when 'RedHat'
        it { is_expected.to contain_package('openldap-servers').with(
          {
            :ensure => :present,
            :name   => 'openldap-servers',
          }
        ) }
      end
    end
  end
end
```

Then, we launch it again:

Now it works with 2 tests marked as pending that we will fix as soon as rspec-puppet is fixed.

## LAUNCH THE ACCEPTANCE TESTS

Finally, we can launch our acceptance tests again and check that everything is still OK:

### ON REDHAT 7

```shell
BEAKER_set=centos-7-x86_64-vagrant bundle exec rspec spec/acceptance/openldap__server_spec.rb
...
openldap::server
  running puppet code
localhost $ scp /tmp/beaker20150305-11384-7chojc centos-7-x64:/tmp/apply_manifest.pp.XqY8IJ {:ignore => }
localhost $ scp /tmp/beaker20150305-11384-1opmurv centos-7-x64:/tmp/apply_manifest.pp.hLyO5k {:ignore => }
    should work with no errors
    Port "389"
      should be listening
    Service "slapd"
      should be enabled
      should be running
Destroying vagrant boxes
==> centos-7-x64: Forcing shutdown of VM...
==> centos-7-x64: Destroying VM and associated drives...
 
Finished in 19.05 seconds (files took 2 minutes 17.4 seconds to load)
4 examples, 0 failures
```

### ON DEBIAN 7

```shell
BEAKER_set=debian-7-x86_64-vagrant bundle exec rspec spec/acceptance/openldap__server_spec.rb
...
openldap::server
  running puppet code
localhost $ scp /tmp/beaker20150305-13082-1sffkj5 debian-7-x64:/tmp/apply_manifest.pp.5AjHHb {:ignore => }
localhost $ scp /tmp/beaker20150305-13082-9tjbgg debian-7-x64:/tmp/apply_manifest.pp.1HcVFQ {:ignore => }
    should work with no errors
    Port "389"
      should be listening
    Service "slapd"
      should be enabled
      should be running
Destroying vagrant boxes
==> debian-7-x64: Forcing shutdown of VM...
==> debian-7-x64: Destroying VM and associated drives...
 
Finished in 17.89 seconds (files took 1 minute 31.4 seconds to load)
4 examples, 0 failures
```

We have now seen how to use unit and acceptance tests to help limit the risks when refactoring our code. Next time, we’ll go a little bit further by developing a custom type and provider to manage OpenLDAP databases in a behavior/test driven manner.
