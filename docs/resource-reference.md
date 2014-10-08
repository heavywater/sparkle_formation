## Resource Reference

This is a best-effort list of commonly used cloudformation resources and their
availability/feature set in supported cloud providers. This list does
not cover everything that may be offered by a specific provider. 

#### Auto Scaling Groups

|Provider    |Available |Caveats & Limitations                                                        |
|------------|----------|-----------------------------------------------------------------------------|
|AWS         |Yes       |EC2 Classic and VPC use mutually exclusive properties.                       |
|Rackspace   |Yes       |Autoscaling will not replace lost instances, only does policy based scaling. |

#### Compute Instances

|Provider    |Available |Caveats & Limitations                                                        |
|------------|----------|-----------------------------------------------------------------------------|
|AWS         |Yes       |                                                                             |
|Rackspace   |Yes       |                                                                             |

#### Load Balancers

|Provider    |Available |Caveats & Limitations                                                        |
|------------|----------|-----------------------------------------------------------------------------|
|AWS         |Yes       |Security Group Ingress is not automatic. Must be defined separately.         |
|Rackspace   |Yes       |Only allows a single LB port & a single instance port.                       |

#### Security Groups
|Provider    |Available |Caveats & Limitations                                                        |
|------------|----------|-----------------------------------------------------------------------------|
|AWS         |Yes       |                                                                             |
|Rackspace   |No        |                                                                             |

#### Storage

|Provider    |Available |Caveats & Limitations                                                        |
|------------|----------|-----------------------------------------------------------------------------|
|AWS         |Yes       |                                                                             |        
|Rackspace   |Yes       |                                                                             |


#### Stack Users
|Provider    |Available |Caveats & Limitations                                                        |
|------------|----------|-----------------------------------------------------------------------------|
|AWS         |Yes       |                                                                             |
|Rackspace   |No        |                                                                             |
        

