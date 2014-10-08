## Intrinsic Functions
The following are all intrinsic AWS Cloudformation functions that are
supported with special syntax in SparkleFormation. Note that these may
not be implemented for all providers. 

### Ref
Ref allows you to reference parameter and resource values. We did this
earlier with the autoscaling group size:
```ruby
parameters.web_nodes do
  type 'Number'
  description 'Number of web nodes for ASG.'
  default '2'
end

...

min_size ref!(:web_nodes)
```
It also works for resource names. The following returns the name of
the launch configuration so we can use it in the autoscaling group
properties. 
```ruby
ref!(:website_launch_config)
```  

### Join
A Join combines strings. You can use Refs and Mappings within a Join.
```ruby
join!(ref!(:environment), '-', map!(:region_map, ref!('AWS::Region'), :ami))
```
Would return 'development-us-east-1', if we built a stack in the
AWS  Virgnia region and provided 'development' for the environment
parameter. 

### Attr
Certain resources attributes can be retrieved directly. To access an
IAM user's (in this case, :cfn_user) secret key:
```ruby
attr!(:cfn_user, :secret_access_key)
```
