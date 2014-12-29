## SparkleFormation Building Blocks

Using SparkleFormation for the above template has already saved us
many keystrokes, but what about reusing SparkleFormation code between
similar stacks? This is where SparkleFormation concepts like
components, dynamics, and registries come into play.

### Directory Structure

SparkleFormation Files should be stored in a `cloudformation`
directory with the following subdirectory structure:

```sh
cloudformation
└───components
└───dynamics
└───registry
└───templates
```

The `components`, `dynamics`, and `registry` directories each contatin
a specific type of SparkleFormation building block. In addition to
these directories, you should have at least one directory for
templates (`template`, in the above example). You may have as many
template directories as you need, and these can be used to organize
SparkleFormation templates by function, ownership group, or other
classification system, as in:

```sh
cloudformation
└───application
└───components
└───database
└───dynamics
└───registry
```

or 

```sh
cloudformation
└───canadian_team
└───components
└───dynamics
└───registry
└───us_team
```

### Components

Components are static configuration which can be reused between many
stack templates. In our example case we have decided that all our
stacks will need to make use of IAM credentials, so we will create
a component which allows us to insert the two IAM resources into any
template in a resuable fashion. The component, which we will call
'base' and put in a file called 'base.rb,' would look like this:

```ruby
SparkleFormation.build do
  set!('AWSTemplateFormatVersion', '2010-09-09')

  resources.cfn_user do
    type 'AWS::IAM::User'
    properties.path '/'
    properties.policies _array(
      -> {
        policy_name 'cfn_access'
        policy_document.statement _array(
          -> {
            effect 'Allow'
            action 'cloudformation:DescribeStackResource'
            resource '*' 
          }
        )
      }
    )
  end

  resources.cfn_keys do
    type 'AWS::IAM::AccessKey'
    properties.user_name ref!(:cfn_user)
  end
end
```

After moving these resources out of the initial template and into a
component, we will update the template so that the base component is
loaded on the first line, and the resources it contains are no longer
present in the template itself:

```ruby
SparkleFormation.new('website').load(:base).overrides do

  description 'Supercool Website'

  parameters.web_nodes do
    type 'Number'
    description 'Number of web nodes for ASG.'
    default 2
  end

  resources.website_autoscale do
    type 'AWS::AutoScaling::AutoScalingGroup'
    properties do
      availability_zones({'Fn::GetAZs' => ''})
      launch_configuration_name ref!(:website_launch_config)
      min_size ref!(:web_nodes)
      max_size ref!(:web_nodes)
    end
  end

  resources.website_launch_config do
    type 'AWS::AutoScaling::LaunchConfiguration'
    properties do
      security_groups [ ref!(:security_group_website) ]
      key_name 'sparkleinfrakey'
      image_id 'ami-59a4a230'
      instance_type 'm3.medium'
    end
  end

  resources.website_elb do
    type 'AWS::ElasticLoadBalancing::LoadBalancer'
    properties do
      availability_zones._set('Fn::GetAZs', '')
      listeners _array(
        -> {
          load_balancer_port '80'
          protocol 'HTTP'
          instance_port '80'
          instance_protocol 'HTTP'
        }
      )
      health_check do
        target 'HTTP:80/'
        healthy_threshold '3'
        unhealthy_threshold '3'
        interval '10'
        timeout '8'
      end
    end
  end
end
```

### Dynamics

Like components, dynamics are another SparkleFormation feature which
enables code reuse between stack templates. Where components are
static, dynamics are useful for creating unique resources via
the passing of arguments.

In our example scenario we have decided that we want to use elastic
load balancer resources in many of our stack templates, we want to
create a dynamic which makes inserting ELB resources much easier than
copying the full resource configuration between templates.

The resulting implementation would look something like this:

```ruby
SparkleFormation.dynamic(:elb) do |_name, _config={}|
    resources.("#{_name}_elb".to_sym) do
      type 'AWS::ElasticLoadBalancing::LoadBalancer'
      properties do
        availability_zones._set('Fn::GetAZs', '')
        listeners _array(
          -> {
            load_balancer_port _config[:load_balancer_port] || '80'
            protocol _config[:protocol] || 'HTTP'
            instance_port _config[:instance_port] || '80'
            instance_protocol _config[:instance_protocol] || 'HTTP'
          }
        )
        health_check do
          target _config[:target] || 'HTTP:80/'
          healthy_threshold _config[:healthy_threshold] || '3'
          unhealthy_threshold _config[:unhealthy_threshold] || '3'
          interval _config[:interval] || '10'
          timeout _config[:timeout] || '8'
        end
      end
  end
end
```

This dynamic accepts two arguments: `_name` (a string, required) and `_config`
(a hash, optional). The dynamic will use the values passed in these
arguments to generate a new ELB resource, and override the default ELB
properties wherever a corresponding key/value pair is provided in the
`_config` hash.

Once updated to make use of the new ELB dynamic, our template looks
like this:

```ruby
SparkleFormation.new('website').load(:base).overrides do

  set!('AWSTemplateFormatVersion', '2010-09-09')

  description 'Supercool Website'

  parameters.web_nodes do
    type 'Number'
    description 'Number of web nodes for ASG.'
    default 2
  end

  resources.security_group_website do
    type 'AWS::EC2::SecurityGroup'
    properties do
      group_description 'Enable SSH'
      security_group_ingress array!(
        -> {
          ip_protocol 'tcp'
          from_port 22
          to_port 22
          cidr_ip '0.0.0.0/0'
        }
      )
    end
  end

  resources.website_autoscale do
    type 'AWS::AutoScaling::AutoScalingGroup'
    properties do
      availability_zones({'Fn::GetAZs' => ''})
      launch_configuration_name ref!(:website_launch_config)
      min_size ref!(:web_nodes)
      max_size ref!(:web_nodes)
    end
  end

  resources.website_launch_config do
    type 'AWS::AutoScaling::LaunchConfiguration'
    properties do
      security_groups [ ref!(:security_group_website) ]
      key_name 'sparkleinfrakey'
      image_id 'ami-59a4a230'
      instance_type 'm3.medium'
    end
  end

  dynamic!(:elb, 'website')
end
```

If we wanted to override the default configuration for the ELB,
e.g. to configure the ELB to listen on and communicate with back-end
node on port 8080 instead of 80, we can specify these override values
in the configuration passed to the ELB dynamic:

```ruby
  dynamic!(:elb, 'website', :load_balancer_port => 8080, :instance_port => 8080)
```

We're passing three arguments here:

1. `:elb` is the name of the dynamic to insert, as a ruby symbol
2. A name string (`'website'`), which is passed to `_name` in the
dynamic. This is prepended to the resource name in the dynamic,
resulting in a unique resource name.
3. The ELB ports to configure as key/value pairs. These are passed into the dynamic as
the `_config` hash.

### Registries

Similar to dynamics, registries are reusable resource configuration
code which can be reused inside different resource definitions.

Registries are useful for defining properties that may be reused
between resources of different types. For example, the
LaunchConfiguration and Instance resource types make use of metadata
properties which inform both resource types how to bootstrap one or
more instances.

In our example scenario we would like our new instances to run
'sudo apt-get update && sudo apt-get upgrade -y' at first boot,
regardless of whether or not the instances are members of an
autoscaling group.

Because these resources are of different types, placing the metadata
required for this task directly inside a dynamic isn't going to work
quite the way we need. Instead we can put this inside a registry which
can be inserted into the resources defined in one or more dynamics:

```ruby
SparkleFormation::Registry.register(:apt_get_update) do
  metadata('AWS::CloudFormation::Init') do
    _camel_keys_set(:auto_disable)
    config do
      commands('01_apt_get_update') do
        command 'sudo apt-get update'
      end
      commands('02_apt_get_upgrade') do
        command 'sudo apt-get upgrade -y'
      end
    end
  end
end
```

What is `_camel_keys_set`? Since SparkleFormation is just transforming Ruby hashes from snake case to camel case JSON hashes, `_camel_keys_set` works out the proper casing for this registry to make sure it's rendered as proper JSON. Now we can insert this registry entry into our existing template, to
ensure that apt is updated upon provisioning:

```ruby
SparkleFormation.new(:website).load(:base).overrides do

  description 'Supercool Website'

  parameters.web_nodes do
    type 'Number'
    description 'Number of web nodes for ASG.'
    default 2
  end

  resources.website_autoscale do
    type 'AWS::AutoScaling::AutoScalingGroup'
    properties do
      availability_zones({'Fn::GetAZs' => ''})
      launch_configuration_name ref!(:website_launch_config)
      min_size ref!(:web_nodes)
      max_size ref!(:web_nodes)
    end
  end

  resources.website_launch_config do
    type 'AWS::AutoScaling::LaunchConfiguration'
    registry!(:apt_get_update, 'website')
    properties do
      image_id 'ami-123456'
      instance_type 'm3.medium'
    end
  end

  dynamic!(:elb, 'website')

end
```
