# What do we start with?
We start with a rather common use case that the entire industry has: a service that provides an API and potentially a CLI for alternative operations. In this case, we have what we called: services-orders. It is a Java service built with Maven that contains two components:
* A CLI that has one-of commands
* An API for serving tiendanube's orders.

This is deployed in kubernetes using CircleCI and a very simple helm chart that simply groups together a few resources (service, deployment, ingress, etc). There are two components that get deployed:
* **Deployment+Service+Ingress**: deployed using a helm chart that is stored on s3.
* **CronJob**: manually deployed using kubectl and a yml file that lives on the repo itself.

The only "special" thing the service's pipeline has is the ability to deploy feature branches. This workflow runs on every single PR but there is an approval process that stops everything from starting. In this workflow, we use the helm chart and replace the corresponding values of the deployment, service and ingress so that we have a brand new service running in parallel. Given how the helm chart is setup, you can actually add this feature branch to the list of pods that will receive production traffic so it can potentially be used as a manual canary release process. Where you first deploy the feature branch with a small number of pods (maybe 10% of the total pods available), validate that everything works correctly then merge the feature branch and deploy the prod the brand new code.

The workflows for each environment are very similar, they all do a variation of:
1. Build and Test
2. Run internal security vulnerability tools
3. Bulid and push image to ECR
4. Wait for manual approval
5. Deploy to EKS
6. Notify of deployment via Slack

### What do we need/like from this setup?
We definitely want to keep the ability to wait for approvals before moving forward with deploys. This allows developer to just push everything without any impact to production or staging resources. It is particularly useful when developing in feature branches, you not always want a feature branch deployment and even if you do you might not want it for every single deploy.

### What don't we like from this setup?
The entire process is fairly straightforward with the exception of the helm chart and how things are specified there. This is more a problem I have with helm and how we use it than a problem of the pipeline itself. I don't like when the entire definition is done with yaml and a non-expressive API. For example on the feature branch deploy we send the following "arguments" to a runner that wraps a custom image built inhouse:
```yaml
  args: |
    --set nameOverride=${CIRCLE_PROJECT_REPONAME}-${CIRCLE_BRANCH} \
    --set fullnameOverride=${CIRCLE_PROJECT_REPONAME}-${CIRCLE_BRANCH} \
    --set labels.app=${CIRCLE_PROJECT_REPONAME}-${CIRCLE_BRANCH} \
    --set labels.service=${CIRCLE_PROJECT_REPONAME} \
    --set-string labels.branch=${CIRCLE_BRANCH} \
    --set-string labels.commit_id=${CIRCLE_SHA1:0:7} \
    --set-string labels.protected_branch="false" \
    --set-string labels.version=${CIRCLE_BRANCH} \
    --set-string env.APP_ENDPOINT="https://${CIRCLE_PROJECT_REPONAME}.${CIRCLE_BRANCH}.<url>.com" \
    --set ingress.hosts[0]=${CIRCLE_PROJECT_REPONAME}.${CIRCLE_BRANCH}.<url>.com \
    --set-string labels."tags\.datadoghq\.com/version"=${CIRCLE_SHA1:0:7} \
    --set env.DD_VERSION=${CIRCLE_SHA1:0:7} \
```

If I'm a developer writing the pipeline for the application that I own and operate I want to know at first glance: i) what impact each of those arguments have? Potentially provided with docs or something built into the LSP; ii) what other options do I have available?; iii) what kind of observability will this tool give me through the deployment process?

The entire process, however simple, is complicated to fully understand what each step does. Since everything is an image with custom logic, if a developer wants to debug and understand what is happening they have to jump from repo to repo to find the code that is implemented on each step and triage each thing separately.

### How frequently do we find ourselves changing this pipeline?
Honestly, very rarely. When you work on services such as this one it is rare that you have the need to change the process that controls the entire lifecycle of the application. If changes are necessary, they are usually very specific like adding new labels to a kubernetes resource.

The one change that in this particular case might be more frequent is the need to add new CronJob resources. This service is the owner of a critical resource (orders) within TiendaNube and as such there are quite a few processes that we have to execute on a schedule. This processes operate over the database that has all orders and because of that we believe it is extremely necessary to hold the logic that performs those operations as close to the logic of the application itself. This is so that we make sure that we can control all operations that are done on orders in a single place. Facilitating observability and ownership.

# How would this pipeline work with Dagger?
Quick refresher of the entire workflow:
1. Build and Test
2. Run internal security vulnerability tools
3. Bulid and push image to ECR
4. Wait for manual approval
5. Deploy to EKS
6. Notify of deployment via Slack

We start by identifying which steps of this workflow *would not* be implemented with Dagger since it is not the tool for the job:
* **Wait for manual approval**: can it be implemented with Dagger? Probably, but it won't really give us any benefits. We believe it is better to have a good UI already in place where developers can see what is being executed and simply hit "Approve".
* **Notify deployment via slack**: can it be implemented with Dagger? Definitely. However, CI systems such as CircleCI and Github Actions already have a set of niceties implemented that make it extremely simple for developers to add in steps such as this one. It is simpler, in my opinion, to add a call to a docker image in the yaml workflow than to add code to the Dagger pipeline that does this. We could potentially run the same image from the Dagger code itself but I don't personally see that as an improvement/benefit.

The steps were Dagger would be a good fit are:
* **Build and Test**
* **Run internal security vulnerability tools**
* **Bulid and push image to ECR**
* **Deploy to EKS**

With this separation, we can already see that Dagger would not really be the "worfklow executor". This means that it wouldn't really be the owner of the "dag" that controls the entire lifecycle of the application. That, instead, would be left for CircleCI and the existing yaml. This means that the changes we would have to make are:
* Implement each step using a Dagger SDK
* Change workflow to call Dagger instead on each step

### What benefits did Dagger give us?
* **Better developer ergonomics**: the build and test process in this particular case is rather simple. However, they often get more complicated with external dependencies (such as an actual database or cache) and different kinds of tests. In some cases, these might need to be built for multiple architectures. In those cases, the fact that everything with Dagger is code and everything is built with the SDK I believe it would be better to make a program more extensive with additional logic than to have that on a yaml file. This is for two reasons: i) simplicity: if you've ever seen a CI workflow that has logic inside implemented with yaml then you know what I mean; ii) reproducibility: it is just code, the developer writing this code can easily test it locally.
* (Theoretical) **Clear APIs for what can be changed in a deployment**: a complaint I had before was related to having an interface of what I can change on a service in pure yaml. With Dagger, developers could theoretically write a library with well defined types that provide functions to perform the operations used here such as deploying services or cron jobs. This way we would minimize surprises and it would be easy for developers to quickly understand what capabilities the current deploying mechanisms have.

Small clarification: we are already caching everything related to maven dependencies so we don't see an advantage from Dagger's caching mechanisms there.

### What don't we like from this new setup?
We still have some level of separation between orchestration and implementation. It is better than before since the "relevant" pieces of the pipeline are all in the code, you still have to look at two different places: the yaml workflow and the actual Dagger code.

# Does it make sense to migrate existing workflows?
I don't see a big value add to justify migrating this workflows in a stage such as this one.

# Does it make sense to use Dagger for new services?
Maybe. I think it depends on the know how of the team that is creating the service or alternatively what tools a potential SRE team has already developed. I personally prefer to try and follow convention over configuration for this types of problems. If an SRE team provides a tool that simplifies the process of creating a new service, including the pipelines themselves, then developers wouldn't need to do much in the first place. However, if they need to make changes to the process for a new use case you never want SRE teams to be the blockers. For that, having the pipelines be code that can be easily changed by developers would be an advantage.
