### Structure
This task consists of the following repositories:
- this one, containing a single HTML page and a CircleCI config which creates a Docker image and deploys to an ECS Cluster
- https://github.com/Wills287/terraform-modules, a collection of modules which can be combined to set up infrastructure
- https://github.com/Wills287-task/sa-task-infrastructure, the 'live' infrastructure for this task, using the modules above

### ALB URL
The HTML page can be seen at the following URL: http://dev-hello-world-1422859435.eu-west-2.elb.amazonaws.com/

Any updates to this repository will result in a new image being built and deployed. Changes will be visible at the above URL.

### Report
The terraform-modules repo consists of some code that I was playing around with a long time ago, when initially getting familiar with terraform. Several online sources recommended that the best way of using terraform was to set up generic modules that can be re-used to bring up infrastucture in a consistent way. The repo contains code inspired by several open-source terraform modules. I have added some additional modules to complete this task.

Although the task mentions setting up a VPC and subnets is not necessary, the terraform-modules repo already contained code to make this simple, so it has been used as part of the solution.

The sa-task-infrastucture repo combines several modules together, often sending the output of one module to another in order to connect everything together. Everything is combined in a single file, which admittedly isn't the best idea, but it does mean that a single "terraform apply" and "terraform destroy" handles the entire setup and teardown of the infrastucture.

Finally, this repository contains a CircleCI configuration consisting of jobs which:
- build and push a Docker image
- stop the currently running task
- deploy the new image to the ECS Cluster/Service

### Drawbacks
Some parts are a little "hacky". For example, a single host port is used for the task definition which means that a second container cannot run on the same host as they'd occupy the same port. An ideal solution would allow for dynamic host ports, enabling several containers to run on the same host. This is why there is a specific "stop-task" job in CircleCI, as the new container cannot come up unless the previous one has been stopped.

It seems the "intended" way to use ECS with EC2 autoscaling groups is to set up a Capacity Provider and allow ECS to handle the scaling of the instances, which is something I only found out after mostly completing the task.

Everything is currently running in public subnets. In an ideal situation, the EC2 container instances would run in private subnets and only allow traffic from the load balancer placed in a public subnet. This has been done because ECS requires instances to have external network access to communicate with the Amazon ECS service endpoint (https://docs.aws.amazon.com/AmazonECS/latest/developerguide/create-public-private-vpc.html). To allow access from a private subnet, a NAT Gateway would be required, which costs money.

### Questions
- How would you handle autoscaling at a server (ec2) level?

To a certain extent this is partially already handled by the ASG module, which sets up CloudWatch Alarms to autoscale when CPU utilisation is above or below certain thresholds for a period of time. This could be extended by also monitoring memory usage with alarms for scaling in a similar way.

ECS also has a "Managed scaling" configuration when setting up a Capacity Provider which may be able to handle the scaling, but I've not used this and cannot say how useful it is.

Previously, I have used a cluster autoscaler deployment for handling EC2 instance scaling when running on Kubernetes.
- How would you handle autoscaling at a container (ECS) level?

Within ECS, there is functionality to scale services (https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-auto-scaling.html). One possible way of scaling would be to use a Target Tracking scaling policy, which aims to keep a metric, such as average CPU utilisation, at a set value. If CPU usage goes above the given value, the number of containers scales up and once CPU usage drops below the value, the number of containers scales down.

Within Kubernetes, I have previously used a Horizontal Pod Autoscaler (https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) to scale up or down.
- How would you handle logging?

Ideally, Fluentd or Logstash running on an instance, collecting logs from containers/pods and pushing them to Elasticsearch. Not entirely sure how this can be configured within ECS, but I have done this with Kubernetes using a Daemonset to run one logger per node and collect logs.
- How would you handle observability of your services?

Set up metrics gathering using Prometheus or a similar tool. Set up tracing instrumentation using a tool like Zipkin or New Relic. I have some familiarity with the tools mentioned here as I've used them when developing Java applications, particularly New Relic for investigating latency issues.
- How would you handle authentication at a code and infrastructure level?

At an infrastructure level, IAM can be used to control who can create, edit or delete infrastructure. Best practice would be to use the principle of least privilege and grant the exact access required and no more.

IAM can also integrate with Kubernetes Service Accounts, which allows restriction of permissions at the pod level if applications require access to AWS services.

At the code level, an application like Keycloak can integrate with your applications and allow role-based access for users. For example, we currently have Keycloak set up to pass along a JWT which defines a user and their roles. The application then filters requests to check whether the user has permissions to do whatever they're trying to do.
