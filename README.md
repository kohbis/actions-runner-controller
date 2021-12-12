# actions-runner-controller

[![awesome-runners](https://img.shields.io/badge/listed%20on-awesome--runners-blue.svg)](https://github.com/jonico/awesome-runners)

This controller operates self-hosted runners for GitHub Actions on your Kubernetes cluster.

ToC:

- [Motivation](#motivation)
- [Installation](#installation)
  - [GitHub Enterprise Support](#github-enterprise-support)
- [Setting Up Authentication with GitHub API](#setting-up-authentication-with-github-api)
  - [Deploying Using GitHub App Authentication](#deploying-using-github-app-authentication)
  - [Deploying Using PAT Authentication](#deploying-using-pat-authentication)
- [Deploying Multiple Controllers](#deploying-multiple-controllers)  
- [Usage](#usage)
  - [Repository Runners](#repository-runners)
  - [Organization Runners](#organization-runners)
  - [Enterprise Runners](#enterprise-runners)
  - [RunnerDeployments](#runnerdeployments)
  - [Autoscaling](#autoscaling)
    - [Anti-Flapping Configuration](#anti-flapping-configuration)
    - [Pull Driven Scaling](#pull-driven-scaling)
    - [Webhook Driven Scaling](#webhook-driven-scaling)
    - [Autoscaling to/from 0](#autoscaling-tofrom-0)
    - [Scheduled Overrides](#scheduled-overrides)
  - [Runner with DinD](#runner-with-dind)
  - [Additional Tweaks](#additional-tweaks)
  - [Runner Labels](#runner-labels)
  - [Runner Groups](#runner-groups)
  - [Using IRSA (IAM Roles for Service Accounts) in EKS](#using-irsa-iam-roles-for-service-accounts-in-eks)
  - [Stateful Runners](#stateful-runners)
  - [Ephemeral Runners](#ephemeral-runners)
  - [Software Installed in the Runner Image](#software-installed-in-the-runner-image)
  - [Using without cert-manager](#using-without-cert-manager)
  - [Common Errors](#common-errors)
- [Contributing](#contributing)

## Motivation

[GitHub Actions](https://github.com/features/actions) is a very useful tool for automating development. GitHub Actions jobs are run in the cloud by default, but you may want to run your jobs in your environment. [Self-hosted runner](https://github.com/actions/runner) can be used for such use cases, but requires the provisioning and configuration of a virtual machine instance. Instead if you already have a Kubernetes cluster, it makes more sense to run the self-hosted runner on top of it.

**actions-runner-controller** makes that possible. Just create a *Runner* resource on your Kubernetes, and it will run and operate the self-hosted runner for the specified repository. Combined with Kubernetes RBAC, you can also build simple Self-hosted runners as a Service.

## Installation

By default, actions-runner-controller uses [cert-manager](https://cert-manager.io/docs/installation/kubernetes/) for certificate management of Admission Webhook. Make sure you have already installed cert-manager before you install. The installation instructions for cert-manager can be found below.

- [Installing cert-manager on Kubernetes](https://cert-manager.io/docs/installation/kubernetes/)

Subsequent to this, install the custom resource definitions and actions-runner-controller with `kubectl` or `helm`. This will create actions-runner-system namespace in your Kubernetes and deploy the required resources.

**Kubectl Deployment:**

```shell
# REPLACE "v0.20.2" with the version you wish to deploy
kubectl apply -f https://github.com/actions-runner-controller/actions-runner-controller/releases/download/v0.20.2/actions-runner-controller.yaml
```

**Helm Deployment:**

Configure your values.yaml, see the chart's [README](./charts/actions-runner-controller/README.md) for the values documentation

```shell
helm repo add actions-runner-controller https://actions-runner-controller.github.io/actions-runner-controller
helm upgrade --install --namespace actions-runner-system --create-namespace \
             --wait actions-runner-controller actions-runner-controller/actions-runner-controller
```

### GitHub Enterprise Support

The solution supports both GHEC (GitHub Enterprise Cloud) and GHES (GitHub Enterprise Server) editions as well as regular GitHub. Both PAT (personal access token) and GitHub App authentication works for installations that will be deploying either repository level and / or organization level runners. If you need to deploy enterprise level runners then you are restricted to PAT based authentication as GitHub doesn't support GitHub App based authentication for enterprise runners currently.

If you are deploying this solution into a GHES environment then you will need version >= [3.0.0](https://docs.github.com/en/enterprise-server@3.0/admin/release-notes) as a minimum, in order to use all the features of actions-runner-controller >= [3.3.0](https://docs.github.com/en/enterprise-server@3.3/admin/release-notes) is required.

When deploying the solution for a GHES environment you need to provide an additional environment variable as part of the controller deployment:

```shell
kubectl set env deploy controller-manager -c manager GITHUB_ENTERPRISE_URL=<GHEC/S URL> --namespace actions-runner-system
```

**_Note: The repository maintainers do not have an enterprise environment (cloud or server). Support for the enterprise specific feature set is community driven and on a best effort basis. PRs from the community are welcomed to add features and maintain support._**

## Setting Up Authentication with GitHub API

There are two ways for actions-runner-controller to authenticate with the GitHub API (only 1 can be configured at a time however):

1. Using a GitHub App (not supported for enterprise level runners due to lack of support from GitHub)
2. Using a PAT

Functionality wise, there isn't much of a difference between the 2 authentication methods. The primarily benefit of authenticating via a GitHub App is an [increased API quota](https://docs.github.com/en/developers/apps/rate-limits-for-github-apps).

If you are deploying the solution for a GHES environment you are able to [configure your rate limit settings](https://docs.github.com/en/enterprise-server@3.0/admin/configuration/configuring-rate-limits) making the main benefit irrelevant. If you're deploying the solution for a GHEC or regular GitHub environment and you run into rate limit issues, consider deploying the solution using the GitHub App authentication method instead.

### Deploying Using GitHub App Authentication

You can create a GitHub App for either your user account or any organization, below are the app permissions required for each supported type of runner:

_Note: Links are provided further down to create an app for your logged in user account or an organization with the permissions for all runner types set in each link's query string_

**Required Permissions for Repository Runners:**<br />
**Repository Permissions**

* Actions (read)
* Administration (read / write)
* Checks (read) (if you are going to use [Webhook Driven Scaling](#webhook-driven-scaling))
* Metadata (read)

**Required Permissions for Organization Runners:**<br />
**Repository Permissions**

* Actions (read)
* Metadata (read)

**Organization Permissions**
* Self-hosted runners (read / write)

_Note: All API routes mapped to their permissions can be found [here](https://docs.github.com/en/rest/reference/permissions-required-for-github-apps) if you wish to review_

**Subscribe to events**

At this point you have a choice of configuring a webhook, a webhook is needed if you are going to use [webhook driven scaling](#webhook-driven-scaling). The webhook can be configured centrally in the GitHub app itself or separately. In either case the event details are:

* Check run (required for all webhook driven scaling events)
* Workflow job (optionally) (required for [webhook driven scaling with workflow_job events](https://github.com/actions-runner-controller/actions-runner-controller#example-1-scale-on-each-workflow_job-event)

---

**Setup Steps**

If you want to create a GitHub App for your account, open the following link to the creation page, enter any unique name in the "GitHub App name" field, and hit the "Create GitHub App" button at the bottom of the page.

- [Create GitHub Apps on your account](https://github.com/settings/apps/new?url=http://github.com/actions-runner-controller/actions-runner-controller&webhook_active=false&public=false&administration=write&actions=read)

If you want to create a GitHub App for your organization, replace the `:org` part of the following URL with your organization name before opening it. Then enter any unique name in the "GitHub App name" field, and hit the "Create GitHub App" button at the bottom of the page to create a GitHub App.

- [Create GitHub Apps on your organization](https://github.com/organizations/:org/settings/apps/new?url=http://github.com/actions-runner-controller/actions-runner-controller&webhook_active=false&public=false&administration=write&organization_self_hosted_runners=write&actions=read&checks=read)

You will see an *App ID* on the page of the GitHub App you created as follows, the value of this App ID will be used later.

<img width="750" alt="App ID" src="https://user-images.githubusercontent.com/230145/78968802-6e7c8880-7b40-11ea-8b08-0c1b8e6a15f0.png">

Download the private key file by pushing the "Generate a private key" button at the bottom of the GitHub App page. This file will also be used later.

<img width="750" alt="Generate a private key" src="https://user-images.githubusercontent.com/230145/78968805-71777900-7b40-11ea-97e6-55c48dfc44ac.png">

Go to the "Install App" tab on the left side of the page and install the GitHub App that you created for your account or organization.

<img width="750" alt="Install App" src="https://user-images.githubusercontent.com/230145/78968806-72100f80-7b40-11ea-810d-2bd3261e9d40.png">

When the installation is complete, you will be taken to a URL in one of the following formats, the last number of the URL will be used as the Installation ID later (For example, if the URL ends in `settings/installations/12345`, then the Installation ID is `12345`).

- `https://github.com/settings/installations/${INSTALLATION_ID}`
- `https://github.com/organizations/eventreactor/settings/installations/${INSTALLATION_ID}`


Finally, register the App ID (`APP_ID`), Installation ID (`INSTALLATION_ID`), and downloaded private key file (`PRIVATE_KEY_FILE_PATH`) to Kubernetes as Secret.

**Kubectl Deployment:**

```shell
$ kubectl create secret generic controller-manager \
    -n actions-runner-system \
    --from-literal=github_app_id=${APP_ID} \
    --from-literal=github_app_installation_id=${INSTALLATION_ID} \
    --from-file=github_app_private_key=${PRIVATE_KEY_FILE_PATH}
```

**Helm Deployment:**

Configure your values.yaml, see the chart's [README](./charts/actions-runner-controller/README.md) for deploying the secret via Helm

### Deploying Using PAT Authentication

Personal Access Tokens can be used to register a self-hosted runner by *actions-runner-controller*.

Log-in to a GitHub account that has `admin` privileges for the repository, and [create a personal access token](https://github.com/settings/tokens/new) with the appropriate scopes listed below:

**Required Scopes for Repository Runners**

* repo (Full control)

**Required Scopes for Organization Runners**

* repo (Full control)
* admin:org (Full control)
* admin:public_key (read:public_key)
* admin:repo_hook (read:repo_hook)
* admin:org_hook (Full control)
* notifications (Full control)
* workflow (Full control)

**Required Scopes for Enterprise Runners**

* admin:enterprise (manage_runners:enterprise)

_Note: When you deploy enterprise runners they will get access to organizations, however, access to the repositories themselves is **NOT** allowed by default. Each GitHub organization must allow enterprise runner groups to be used in repositories as an initial one time configuration step, this  only needs to be done once after which it is permanent for that runner group._

_Note: GitHub do not document exactly what permissions you get with each PAT scope beyond a vague description. The best documentation they provide on the topic can be found [here](https://docs.github.com/en/developers/apps/building-oauth-apps/scopes-for-oauth-apps) if you wish to review. The docs target OAuth apps and so are incomplete and amy not be 100% accurate._ 

---

Once you have created the appropriate token, deploy it as a secret to your Kubernetes cluster that you are going to deploy the solution on:

**Kubectl Deployment:**

```shell
kubectl create secret generic controller-manager \
    -n actions-runner-system \
    --from-literal=github_token=${GITHUB_TOKEN}
```

**Helm Deployment:**

Configure your values.yaml, see the chart's [README](./charts/actions-runner-controller/README.md) for deploying the secret via Helm

### Deploying Multiple Controllers

> This feature requires controller version => [v0.18.0](https://github.com/actions-runner-controller/actions-runner-controller/releases/tag/v0.18.0)

**_Note: Be aware when using this feature that CRDs are cluster wide and so you should upgrade all of your controllers (and your CRDs) as the same time if you are doing an upgrade. Do not mix and match CRD versions with different controller versions. Doing so risks out of control scaling._**

By default the controller will look for runners in all namespaces, the watch namespace feature allows you to restrict the controller to monitoring a single namespace. This then lets you deploy multiple controllers in a single cluster. You may want to do this either because you wish to scale beyond the API rate limit of a single PAT / GitHub App configuration or you wish to support multiple GitHub organizations with runners installed at the organization level in a single cluster.

This feature is configured via the controller's `--watch-namespace` flag. When a namespace is provided via this flag, the controller will only monitor runners in that namespace.

If you plan on installing all instances of the controller stack into a single namespace you will need to make the names of the resources unique to each stack. In the case of Helm this can be done by giving each install a unique release name, or via the `fullnameOverride` properties.

Alternatively, you can install each controller stack into its own unique namespace (relative to other controller stacks in the cluster), avoiding the need to uniquely prefix resources.

When you go to the route of sharing the namespace while giving each a unique Helm release name, you must also ensure the following values are configured correctly:

- `authSecret.name` needs be unique per stack when each stack is tied to runners in different GitHub organizations and repositories AND you want your GitHub credentials to narrowly scoped.
- `leaderElectionId` needs to be unique per stack. If this is not unique to the stack the controller tries to race onto the leader election lock and resulting in only one stack working concurrently.

## Usage

[GitHub self-hosted runners can be deployed at various levels in a management hierarchy](https://docs.github.com/en/actions/hosting-your-own-runners/about-self-hosted-runners#about-self-hosted-runners):
- The repository level
- The organization level
- The enterprise level

There are two ways to use this controller:

- Manage runners one by one with `Runner`.
- Manage a set of runners with `RunnerDeployment`.

### Repository Runners

To launch a single self-hosted runner, you need to create a manifest file includes `Runner` resource as follows. This example launches a self-hosted runner with name *example-runner* for the *actions-runner-controller/actions-runner-controller* repository.

```yaml
# runner.yaml
apiVersion: actions.summerwind.dev/v1alpha1
kind: Runner
metadata:
  name: example-runner
spec:
  repository: example/myrepo
  env: []
```

Apply the created manifest file to your Kubernetes.

```shell
$ kubectl apply -f runner.yaml
runner.actions.summerwind.dev/example-runner created
```

You can see that the Runner resource has been created.

```shell
$ kubectl get runners
NAME             REPOSITORY                             STATUS
example-runner   actions-runner-controller/actions-runner-controller   Running
```

You can also see that the runner pod has been running.

```shell
$ kubectl get pods
NAME           READY   STATUS    RESTARTS   AGE
example-runner 2/2     Running   0          1m
```

The runner you created has been registered to your repository.

<img width="756" alt="Actions tab in your repository settings" src="https://user-images.githubusercontent.com/230145/73618667-8cbf9700-466c-11ea-80b6-c67e6d3f70e7.png">

Now you can use your self-hosted runner. See the [official documentation](https://help.github.com/en/actions/automating-your-workflow-with-github-actions/using-self-hosted-runners-in-a-workflow) on how to run a job with it.

### Organization Runners

To add the runner to an organization, you only need to replace the `repository` field with `organization`, so the runner will register itself to the organization.

```yaml
# runner.yaml
apiVersion: actions.summerwind.dev/v1alpha1
kind: Runner
metadata:
  name: example-org-runner
spec:
  organization: your-organization-name
```

Now you can see the runner on the organization level (if you have organization owner permissions).

### Enterprise Runners

To add the runner to an enterprise, you only need to replace the `repository` field with `enterprise`, so the runner will register itself to the enterprise.

```yaml
# runner.yaml
apiVersion: actions.summerwind.dev/v1alpha1
kind: Runner
metadata:
  name: example-enterprise-runner
spec:
  enterprise: your-enterprise-name
```

Now you can see the runner on the enterprise level (if you have enterprise access permissions).

### RunnerDeployments

You can manage sets of runners instead of individually through the `RunnerDeployment` kind and its `replicas:` attribute. This kind is required for many of the advanced features.

There are `RunnerReplicaSet` and `RunnerDeployment` kinds that corresponds to the `ReplicaSet` and `Deployment` kinds but for the `Runner` kind.

You typically only need `RunnerDeployment` rather than `RunnerReplicaSet` as the former is for managing the latter.

```yaml
# runnerdeployment.yaml
apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerDeployment
metadata:
  name: example-runnerdeploy
spec:
  replicas: 2
  template:
    spec:
      repository: mumoshu/actions-runner-controller-ci
      env: []
```

Apply the manifest file to your cluster:

```shell
$ kubectl apply -f runnerdeployment.yaml
runnerdeployment.actions.summerwind.dev/example-runnerdeploy created
```

You can see that 2 runners have been created as specified by `replicas: 2`:

```shell
$ kubectl get runners
NAME                             REPOSITORY                             STATUS
example-runnerdeploy2475h595fr   mumoshu/actions-runner-controller-ci   Running
example-runnerdeploy2475ht2qbr   mumoshu/actions-runner-controller-ci   Running
```

### Autoscaling

> Since the release of GitHub's [`workflow_job` webhook](https://docs.github.com/en/developers/webhooks-and-events/webhooks/webhook-events-and-payloads#workflow_job), webhook driven scaling is the preferred way of autoscaling as it enables targeted scaling of your `RunnerDeployment` / `RunnerSet` as it includes the `runs-on` information needed to scale the appropriate runners for that workflow run. More broadly, webhook driven scaling is the preferred scaling option as it is far quicker compared to the pull driven scaling and is easy to setup.

A `RunnerDeployment` or `RunnerSet` (see [stateful runners](#stateful-runners) for more details on this kind) can scale the number of runners between `minReplicas` and `maxReplicas` fields driven by either pull based scaling metrics or via a webhook event (see limitations section of [stateful runners](#stateful-runners) for cavaets of this kind). Whether the autoscaling is driven from a webhook event or pull based metrics it is implemented by backing a `RunnerDeployment` or `RunnerSet` kind with a `HorizontalRunnerAutoscaler` kind.

**_Important!!! If you opt to configure autoscaling, ensure you remove the `replicas:` attribute in the `RunnerDeployment` / `RunnerSet` kinds that are configured for autoscaling [#206](https://github.com/actions-runner-controller/actions-runner-controller/issues/206#issuecomment-748601907)_**

#### Anti-Flapping Configuration

For both pull driven or webhook driven scaling an anti-flapping implementation is included, by default a runner won't be scaled down within 10 minutes of it having been scaled up. This delay is configurable by including the attribute `scaleDownDelaySecondsAfterScaleOut:` in a `HorizontalRunnerAutoscaler` kind's `spec:`.

This configuration has the final say on if a runner can be scaled down or not regardless of the chosen scaling method. Depending on your requirements, you may want to consider adjusting this by setting the `scaleDownDelaySecondsAfterScaleOut:` attribute.

Below is a complete basic example with one of the pull driven scaling metrics.

```yaml
apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerDeployment
metadata:
  name: example-runner-deployment
spec:
  template:
    spec:
      repository: example/myrepo
---
apiVersion: actions.summerwind.dev/v1alpha1
kind: HorizontalRunnerAutoscaler
metadata:
  name: example-runner-deployment-autoscaler
spec:
  # Runners in the targeted RunnerDeployment won't be scaled down
  # for 5 minutes instead of the default 10 minutes now
  scaleDownDelaySecondsAfterScaleOut: 300
  scaleTargetRef:
    name: example-runner-deployment
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: PercentageRunnersBusy
    scaleUpThreshold: '0.75'
    scaleDownThreshold: '0.25'
    scaleUpFactor: '2'
    scaleDownFactor: '0.5'
```

#### Pull Driven Scaling

> To configure webhook driven scaling see the [Webhook Driven Scaling](#webhook-driven-scaling) section

The pull based metrics are configured in the `metrics` attribute of a HRA (see snippet below). The period between polls is defined by the controller's `--sync-period` flag. If this flag isn't provided then the controller defaults to a sync period of 10 minutes. The default value is set to 10 minutes to prevent default deployments rate limiting themselves from the GitHub API, you will most likely want to adjust this.

```yaml
apiVersion: actions.summerwind.dev/v1alpha1
kind: HorizontalRunnerAutoscaler
metadata:
  name: example-runner-deployment-autoscaler
spec:
  scaleTargetRef:
    # Your RunnerDeployment Here
    name: example-runner-deployment
  minReplicas: 1
  maxReplicas: 5
  # Your chosen scaling metrics here
  metrics: [] 
```

**Metric Options:**

**TotalNumberOfQueuedAndInProgressWorkflowRuns**

The `TotalNumberOfQueuedAndInProgressWorkflowRuns` metric polls GitHub for all pending workflow runs against a given set of repositories. The metric will scale the runner count up to the total number of pending jobs at the sync time up to the `maxReplicas` configuration.

**Benefits of this metric**
1. Supports named repositories allowing you to restrict the runner to a specified set of repositories server-side.
2. Scales the runner count based on the depth of the job queue meaning a more 1:1 scaling of runners to queued jobs (caveat, see drawback #4)
3. Like all scaling metrics, you can manage workflow allocation to the RunnerDeployment through the use of [GitHub labels](#runner-labels).

**Drawbacks of this metric**
1. A list of repositories must be included within the scaling metric. Maintaining a list of repositories may not be viable in larger environments or self-serve environments.
2. May not scale quick enough for some users needs. This metric is pull based and so the queue depth is polled as configured by the sync period, as a result scaling performance is bound by this sync period meaning there is a lag to scaling activity.
3. Relatively large amounts of API requests required to maintain this metric, you may run in API rate limit issues depending on the size of your environment and how aggressive your sync period configuration is.
4. The GitHub API doesn't provide a way to filter workflow jobs to just those targeting self-hosted runners. If your environment's workflows target both self-hosted and GitHub hosted runners then the queue depth this metric scales against isn't a true 1:1 mapping of queue depth to required runner count. As a result of this, this metric may scale too aggressively for your actual self-hosted runner count needs.

Example `RunnerDeployment` backed by a `HorizontalRunnerAutoscaler`:

```yaml
apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerDeployment
metadata:
  name: example-runner-deployment
spec:
  template:
    spec:
      repository: example/myrepo
---
apiVersion: actions.summerwind.dev/v1alpha1
kind: HorizontalRunnerAutoscaler
metadata:
  name: example-runner-deployment-autoscaler
spec:
  scaleTargetRef:
    name: example-runner-deployment
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: TotalNumberOfQueuedAndInProgressWorkflowRuns
    repositoryNames:
    - example/myrepo
```

**PercentageRunnersBusy**

The `HorizontalRunnerAutoscaler` will poll GitHub for the number of runners in the `busy` state which live in the RunnerDeployment's namespace, it will then scale depending on how you have configured the scale factors.

**Benefits of this metric**
1. Supports named repositories server-side the same as the `TotalNumberOfQueuedAndInProgressWorkflowRuns` metric [#313](https://github.com/actions-runner-controller/actions-runner-controller/pull/313)
2. Supports GitHub organization wide scaling without maintaining an explicit list of repositories, this is especially useful for those that are working at a larger scale. [#223](https://github.com/actions-runner-controller/actions-runner-controller/pull/223)
3. Like all scaling metrics, you can manage workflow allocation to the RunnerDeployment through the use of [GitHub labels](#runner-labels)
4. Supports scaling desired runner count on both a percentage increase / decrease basis as well as on a fixed increase / decrease count basis [#223](https://github.com/actions-runner-controller/actions-runner-controller/pull/223) [#315](https://github.com/actions-runner-controller/actions-runner-controller/pull/315)

**Drawbacks of this metric**
1. May not scale quick enough for some users needs. This metric is pull based and so the number of busy runners are polled as configured by the sync period, as a result scaling performance is bound by this sync period meaning there is a lag to scaling activity.
2. We are scaling up and down based on indicative information rather than a count of the actual number of queued jobs and so the desired runner count is likely to under provision new runners or overprovision them relative to actual job queue depth, this may or may not be a problem for you.

Examples of each scaling type implemented with a `RunnerDeployment` backed by a `HorizontalRunnerAutoscaler`:

```yaml
---
apiVersion: actions.summerwind.dev/v1alpha1
kind: HorizontalRunnerAutoscaler
metadata:
  name: example-runner-deployment-autoscaler
spec:
  scaleTargetRef:
    name: example-runner-deployment
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: PercentageRunnersBusy
    scaleUpThreshold: '0.75'    # The percentage of busy runners at which the number of desired runners are re-evaluated to scale up
    scaleDownThreshold: '0.3'   # The percentage of busy runners at which the number of desired runners are re-evaluated to scale down
    scaleUpFactor: '1.4'        # The scale up multiplier factor applied to desired count
    scaleDownFactor: '0.7'      # The scale down multiplier factor applied to desired count
```

```yaml
---
apiVersion: actions.summerwind.dev/v1alpha1
kind: HorizontalRunnerAutoscaler
metadata:
  name: example-runner-deployment-autoscaler
spec:
  scaleTargetRef:
    name: example-runner-deployment
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: PercentageRunnersBusy
    scaleUpThreshold: '0.75'    # The percentage of busy runners at which the number of desired runners are re-evaluated to scale up
    scaleDownThreshold: '0.3'   # The percentage of busy runners at which the number of desired runners are re-evaluated to scale down
    scaleUpAdjustment: 2        # The scale up runner count added to desired count
    scaleDownAdjustment: 1      # The scale down runner count subtracted from the desired count
```

#### Webhook Driven Scaling

> To configure pull driven scaling see the [Pull Driven Scaling](#pull-driven-scaling) section

Webhooks are processed by a seperate webhook server. The webhook server receives GitHub Webhook events and scales
[`RunnerDeployments`](#runnerdeployments) by updating corresponding [`HorizontalRunnerAutoscalers`](#autoscaling).

Today, the Webhook server can be configured to respond GitHub `check_run`, `workflow_job`, `pull_request` and `push`  events
by scaling up the matching `HorizontalRunnerAutoscaler` by N replica(s), where `N` is configurable within `HorizontalRunnerAutoscaler`'s `spec:`.

More concretely, you can configure the targeted GitHub event types and the `N` in `scaleUpTriggers`:

```yaml
kind: HorizontalRunnerAutoscaler
spec:
  scaleTargetRef:
    name: example-runners
  scaleUpTriggers:
  - githubEvent:
      checkRun:
        types: ["created"]
        status: "queued"
    amount: 1
    duration: "5m"
```

With the above example, the webhook server scales `example-runners` by `1` replica for 5 minutes on each `check_run` event with the type of `created` and the status of `queued` received.

Of note is the `HRA.spec.scaleUpTriggers[].duration` attribute. This attribute is used to calculate if the replica number added via the trigger is expired or not. On each reconcilation loop, the controller sums up all the non-expiring replica numbers from previous scale up triggers. It then compares the summed desired replica number against the current replica number. If the summed desired replica number > the current number then it means the replica count needs to scale up.

As mentioned previously, the `scaleDownDelaySecondsAfterScaleOut` property has the final say still. If the latest scale-up time + the anti-flapping duration is later than the current time, it doesn’t immediately scale up and instead retries the calculation again later to see if it needs to scale yet.

---

The primary benefit of autoscaling on Webhook compared to the pull driven scaling is that it is far quicker as it allows you to immediately add runners resource rather than waiting for the next sync period.

> You can learn the implementation details in [#282](https://github.com/actions-runner-controller/actions-runner-controller/pull/282)

To enable this feature, you firstly need to install the webhook server, currently, only our Helm chart has the ability install it:
_[see the values documentation for all configuration options](https://github.com/actions-runner-controller/actions-runner-controller/blob/master/charts/actions-runner-controller/README.md)_

```console
$ helm --upgrade install actions-runner-controller/actions-runner-controller \
  githubWebhookServer.enabled=true \
  githubWebhookServer.ports[0].nodePort=33080
```

The above command will result in exposing the node port 33080 for Webhook events. Usually, you need to create an
external loadbalancer targeted to the node port, and register the hostname or the IP address of the external loadbalancer
to the GitHub Webhook.

Once you were able to confirm that the Webhook server is ready and running from GitHub - this is usually verified by the
GitHub sending PING events to the Webhook server - create or update your `HorizontalRunnerAutoscaler` resources
by learning the following configuration examples.

- [Example 1: Scale on each `workflow_job` event](#example-1-scale-on-each-workflow_job-event)
- [Example 2: Scale up on each `check_run` event](#example-2-scale-up-on-each-check_run-event)
- [Example 3: Scale on each `pull_request` event against a given set of branches](#example-3-scale-on-each-pull_request-event-against-a-given-set-of-branches)
- [Example 4: Scale on each `push` event](#example-4-scale-on-each-push-event)

**Note:** All these examples should have **minReplicas** & **maxReplicas** as mandatory parameter even for webhook driven scaling. 

##### Example 1: Scale on each `workflow_job` event

> This feature requires controller version => [v0.20.0](https://github.com/actions-runner-controller/actions-runner-controller/releases/tag/v0.20.0)

_Note: GitHub does not include the runner group information of a repository in the payload of `workflow_job` event in the initial `queued` event. The runner group information is only include for `workflow_job` events when the job has already been allocated to a runner (events with a status of `in_progress` or `completed`). Please do raise feature requests against [GitHub](https://support.github.com/tickets/personal/0) for this information to be included in the initial `queued` event if this would improve autoscaling runners for you._

The most flexible webhook GitHub offers is the `workflow_job` webhook, it includes the `runs-on` information in the payload allowing scaling based on runner labels.

This webhook should cover most people's needs, please experiment with this webhook first before considering the others.

```yaml
kind: RunnerDeployment
metadata:
   name: example-runners
spec:
  template:
    spec:
      repository: example/myrepo
---
kind: HorizontalRunnerAutoscaler
spec:
  scaleTargetRef:
    name: example-runners
  scaleUpTriggers:
  - githubEvent: {}
    duration: "30m"
```

You can configure your GitHub webhook settings to only include `Workflows Job` events, so that it sends us three kinds of `workflow_job` events per a job run.

Each kind has a `status` of `queued`, `in_progress` and `completed`. With the above configuration, `actions-runner-controller` adds one runner for a `workflow_job` event whose `status` is `queued`. Similarly, it removes one runner for a `workflow_job` event whose `status` is `completed`. The cavaet to this to remember is that this the scale down is within the bounds of your `scaleDownDelaySecondsAfterScaleOut` configuration, if this time hasn't past the scale down will be defered.

##### Example 2: Scale up on each `check_run` event

> Note: This should work almost like https://github.com/philips-labs/terraform-aws-github-runner

To scale up replicas of the runners for `example/myrepo` by 1 for 5 minutes on each `check_run`, you write manifests like the below:

```yaml
kind: RunnerDeployment
metadata:
   name: example-runners
spec:
  template:
    spec:
      repository: example/myrepo
---
kind: HorizontalRunnerAutoscaler
spec:
  scaleTargetRef:
    name: example-runners
  scaleUpTriggers:
  - githubEvent:
      checkRun:
        types: ["created"]
        status: "queued"
    amount: 1
    duration: "5m"
```

To scale up replicas of the runners for `myorg` organization by 1 for 5 minutes on each `check_run`, you write manifests like the below:

```yaml
kind: RunnerDeployment
metadata:
   name: example-runners
spec:
  template:
    spec:
      organization: myorg
---
kind: HorizontalRunnerAutoscaler
spec:
  scaleTargetRef:
    name: example-runners
  scaleUpTriggers:
  - githubEvent:
      checkRun:
        types: ["created"]
        status: "queued"
        # Optionally restrict autoscaling to being triggered by events from specific repositories within your organization still
        # repositories: ["myrepo", "myanotherrepo"]
    amount: 1
    duration: "5m"
```

##### Example 3: Scale on each `pull_request` event against a given set of branches

To scale up replicas of the runners for `example/myrepo` by 1 for 5 minutes on each `pull_request` against the `main` or `develop` branch you write manifests like the below:

```yaml
kind: RunnerDeployment
metadata:
   name: example-runners
spec:
  template:
    spec:
      repository: example/myrepo
---
kind: HorizontalRunnerAutoscaler
spec:
  scaleTargetRef:
    name: example-runners
  scaleUpTriggers:
  - githubEvent:
      pullRequest:
        types: ["synchronize"]
        branches: ["main", "develop"]
    amount: 1
    duration: "5m"
```

See ["activity types"](https://docs.github.com/en/actions/reference/events-that-trigger-workflows#pull_request) for the list of valid values for `scaleUpTriggers[].githubEvent.pullRequest.types`.

###### Example 4: Scale on each push event

To scale up replicas of the runners for `example/myrepo` by 1 for 5 minutes on each `push` write manifests like the below:

```yaml
kind: RunnerDeployment
metadata:
   name: example-runners
spec:
  repository: example/myrepo
---
kind: HorizontalRunnerAutoscaler
spec:
  scaleTargetRef:
    name: example-runners
  scaleUpTriggers:
  - githubEvent:
      push:
    amount: 1
    duration: "5m"
```

#### Autoscaling to/from 0

> This feature requires controller version => [v0.19.0](https://github.com/actions-runner-controller/actions-runner-controller/releases/tag/v0.19.0)

_Note: The controller creates a "registration-only" runner per RunnerReplicaSet when it is being scaled to zero and retains it until there are one or more runners available. This is a deprecated feature for GitHub Cloud as "registration-only" runners are no longer needed due to GitHub changing their runner [routing logic](https://docs.github.com/en/enterprise-cloud@latest/actions/hosting-your-own-runners/using-self-hosted-runners-in-a-workflow#routing-precedence-for-self-hosted-runners) to no longer fail a workflow run if it targets a runner label that there are no registered runners for._

The regular `RunnerDeployment` `replicas:` attribute as well as the `HorizontalRunnerAutoscaler` `minReplicas:` attribute supports being set to 0.

The main use case for scaling from 0 is with the `HorizontalRunnerAutoscaler` kind. To scale from 0 whilst still being able to provision runners as jobs are queued we must use the `HorizontalRunnerAutoscaler` with only certain scaling configurations, only the below configurations support scaling from 0 whilst also being able to provision runners as jobs are queued:

- `TotalNumberOfQueuedAndInProgressWorkflowRuns`
- `PercentageRunnersBusy` + `TotalNumberOfQueuedAndInProgressWorkflowRuns`
- `PercentageRunnersBusy` + Webhook-based autoscaling
- Webhook-based autoscaling only

`PercentageRunnersBusy` can't be used alone as, by its definition, it needs one or more GitHub runners to become `busy` to be able to scale. If there isn't a runner to pick up a job and enter a `busy` state then the controller will never know to provision a runner to begin with as this metric has no knowledge of the job queue and is relying using the number of busy runners as a means for calculating the desired replica count.

If a HorizontalRunnerAutoscaler is configured with a secondary metric of `TotalNumberOfQueuedAndInProgressWorkflowRuns` then be aware that the controller will check the primary metric of `PercentageRunnersBusy` first and will only use the secondary metric to calculate the desired replica count if the primary metric returns 0 desired replicas.

#### Scheduled Overrides

> This feature requires controller version => [v0.19.0](https://github.com/actions-runner-controller/actions-runner-controller/releases/tag/v0.19.0)

`Scheduled Overrides` allows you to configure `HorizontalRunnerAutoscaler` so that its `spec:` gets updated only during a certain period of time. This feature is usually used for following scenarios:

- You want to reduce your infrastructure costs by scaling your Kubernetes nodes down outside a given period
- You want to scale for scheduled spikes in workloads

The most basic usage of this feature is to set a non-repeating override:

```yaml
apiVersion: actions.summerwind.dev/v1alpha1
kind: HorizontalRunnerAutoscaler
metadata:
  name: example-runner-deployment-autoscaler
spec:
  scaleTargetRef:
    name: example-runner-deployment
  scheduledOverrides:
  # Override minReplicas to 100 only between 2021-06-01T00:00:00+09:00 and 2021-06-03T00:00:00+09:00
  - startTime: "2021-06-01T00:00:00+09:00"
    endTime: "2021-06-03T00:00:00+09:00"
    minReplicas: 100
  minReplicas: 1
```

A scheduled override without `recurrenceRule` is considered a one-off override, that is active between `startTime` and `endTime`. In the second scenario, it overrides `minReplicas` to `100` only between `2021-06-01T00:00:00+09:00` and `2021-06-03T00:00:00+09:00`.

A more advanced configuration is to include a `recurrenceRule` in the override:

```yaml
apiVersion: actions.summerwind.dev/v1alpha1
kind: HorizontalRunnerAutoscaler
metadata:
  name: example-runner-deployment-autoscaler
spec:
  scaleTargetRef:
    name: example-runner-deployment
  scheduledOverrides:
  # Override minReplicas to 0 only between 0am sat to 0am mon
  - startTime: "2021-05-01T00:00:00+09:00"
    endTime: "2021-05-03T00:00:00+09:00"
    recurrenceRule:
      frequency: Weekly
      # Optional sunset datetime attribute
      # untilTime: "2022-05-01T00:00:00+09:00"
    minReplicas: 0
  minReplicas: 1
```

 A recurring override is initially active between `startTime` and `endTime`, and then it repeatedly get activated after a certain period of time denoted by `frequency`.

`frequecy` can take one of the following values:

- `Daily`
- `Weekly`
- `Monthly`
- `Yearly`

By default, a scheduled override repeats forever. If you want it to repeat until a specific point in time, define `untilTime`. The controller create the last recurrence of the override until the recurrence's `startTime` is equal or earlier than `untilTime`.

Do ensure that you have enough slack for `untilTime` so that a delayed or offline `actions-runner-controller` is much less likely to miss the last recurrence. For example, you might want to set `untilTime` to `M` minutes after the last recurrence's `startTime`, so that `actions-runner-controller` being offline up to `M` minutes doesn't miss the last recurrence.

**Combining Multiple Scheduled Overrides**:

In case you have a more complex scenarios, try writing two or more entries under `scheduledOverrides`.

The earlier entry is prioritized higher than later entries. So you usually define one-time overrides in the top of your list, then yearly, monthly, weekly, and lastly daily overrides.

A common use case for this may be to have 1 override to scale to 0 during the week outside of core business hours and another override to scale to 0 during all hours of the weekend.

### Runner with DinD

When using default runner, runner pod starts up 2 containers: runner and DinD (Docker-in-Docker). This might create issues if there's `LimitRange` set to namespace.

```yaml
# dindrunnerdeployment.yaml
apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerDeployment
metadata:
  name: example-dindrunnerdeploy
spec:
  replicas: 2
  template:
    spec:
      image: summerwind/actions-runner-dind
      dockerdWithinRunnerContainer: true
      repository: mumoshu/actions-runner-controller-ci
      env: []
```

This also helps with resources, as you don't need to give resources separately to docker and runner.

### Additional Tweaks

You can pass details through the spec selector. Here's an eg. of what you may like to do:

```yaml
apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerDeployment
metadata:
  name: actions-runner
  namespace: default
spec:
  replicas: 2
  template:
    metadata:
      annotations:
        cluster-autoscaler.kubernetes.io/safe-to-evict: "true"
    spec:
      nodeSelector:
        node-role.kubernetes.io/test: ""

      securityContext:
        #All level/role/type/user values will vary based on your SELinux policies.
        #See https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux_atomic_host/7/html/container_security_guide/docker_selinux_security_policy for information about SELinux with containers
        seLinuxOptions:
          level: "s0"
          role: "system_r"
          type: "super_t"
          user: "system_u"

      tolerations:
      - effect: NoSchedule
        key: node-role.kubernetes.io/test
        operator: Exists

      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              runner-deployment-name: actions-runner

      repository: mumoshu/actions-runner-controller-ci
      # The default "summerwind/actions-runner" images are available at DockerHub:
      # https://hub.docker.com/r/summerwind/actions-runner
      # You can also build your own and specify it like the below:
      image: custom-image/actions-runner:latest
      imagePullPolicy: Always
      resources:
        limits:
          cpu: "4.0"
          memory: "8Gi"
        requests:
          cpu: "2.0"
          memory: "4Gi"
      # Timeout after a node crashed or became unreachable to evict your pods somewhere else (default 5mins)
      tolerations:
        - key: "node.kubernetes.io/unreachable"
          operator: "Exists"
          effect: "NoExecute"
          tolerationSeconds: 10
      # true (default) = The runner restarts after running jobs, to ensure a clean and reproducible build environment
      # false = The runner is persistent across jobs and doesn't automatically restart
      # This directly controls the behaviour of `--once` flag provided to the github runner
      ephemeral: false
      # true (default) = A privileged docker sidecar container is included in the runner pod.
      # false = A docker sidecar container is not included in the runner pod and you can't use docker.
      # If set to false, there are no privileged container and you cannot use docker.
      dockerEnabled: false
      # Optional Docker containers network MTU
      # If your network card MTU is smaller than Docker's default 1500, you might encounter Docker networking issues.
      # To fix these issues, you should setup Docker MTU smaller than or equal to that on the outgoing network card.
      # More information:
      # - https://mlohr.com/docker-mtu/
      dockerMTU: 1500
      # Optional Docker registry mirror
      # Docker Hub has an aggressive rate-limit configuration for free plans.
      # To avoid disruptions in your CI/CD pipelines, you might want to setup an external or on-premises Docker registry mirror.
      # More information:
      # - https://docs.docker.com/docker-hub/download-rate-limit/
      # - https://cloud.google.com/container-registry/docs/pulling-cached-images
      dockerRegistryMirror: https://mirror.gcr.io/
      # false (default) = Docker support is provided by a sidecar container deployed in the runner pod.
      # true = No docker sidecar container is deployed in the runner pod but docker can be used within the runner container instead. The image summerwind/actions-runner-dind is used by default.
      dockerdWithinRunnerContainer: true
      # Docker sidecar container image tweaks examples below, only applicable if dockerdWithinRunnerContainer = false
      dockerdContainerResources:
        limits:
          cpu: "4.0"
          memory: "8Gi"
        requests:
          cpu: "2.0"
          memory: "4Gi"
      # Additional N number of sidecar containers
      sidecarContainers:
        - name: mysql
          image: mysql:5.7
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: abcd1234
          securityContext:
            runAsUser: 0
      # workDir if not specified (default = /runner/_work)
      # You can customise this setting allowing you to change the default working directory location
      # for example, the below setting is the same as on the ubuntu-18.04 image
      workDir: /home/runner/work
      # You can mount some of the shared volumes to the dind container using dockerVolumeMounts, like any other volume mounting.
      # NOTE: in case you want to use an hostPath like the following example, make sure that Kubernetes doesn't schedule more than one runner
      # per physical host. You can achieve that by setting pod anti-affinity rules and/or resource requests/limits.
      volumes:
        - name: docker-extra
          hostPath:
            path: /mnt/docker-extra
            type: DirectoryOrCreate
        - name: repo
          hostPath:
            path: /mnt/repo
            type: DirectoryOrCreate
      dockerVolumeMounts:
        - mountPath: /var/lib/docker
          name: docker-extra
      # You can mount some of the shared volumes to the runner container using volumeMounts.
      # NOTE: Do not try to mount the volume onto the runner workdir itself as it will not work. You could mount it however on a sub directory in the runner workdir
      # Please see https://github.com/actions-runner-controller/actions-runner-controller/issues/630#issuecomment-862087323 for more information.
      volumeMounts:
        - mountPath: /home/runner/work/repo
          name: repo
      # Optional storage medium type of runner volume mount.
      # More info: https://kubernetes.io/docs/concepts/storage/volumes/#emptydir
      # "" (default) = Node's default medium
      # Memory = RAM-backed filesystem (tmpfs)
      # NOTE: Using RAM-backed filesystem gives you fastest possible storage on your host nodes.
      volumeStorageMedium: ""
      # Total amount of local storage resources required for runner volume mount.
      # The default limit is undefined.
      # NOTE: You can make sure that nodes' resources are never exceeded by limiting used storage size per runner pod.
      # You can even disable the runner mount completely by setting limit to zero if dockerdWithinRunnerContainer = true.
      # Please see https://github.com/actions-runner-controller/actions-runner-controller/pull/674 for more information.
      volumeSizeLimit: 4Gi
      # Optional name of the container runtime configuration that should be used for pods.
      # This must match the name of a RuntimeClass resource available on the cluster.
      # More info: https://kubernetes.io/docs/concepts/containers/runtime-class
      runtimeClassName: "runc"
```

### Runner Labels

To run a workflow job on a self-hosted runner, you can use the following syntax in your workflow:

```yaml
jobs:
  release:
    runs-on: self-hosted
```

When you have multiple kinds of self-hosted runners, you can distinguish between them using labels. In order to do so, you can specify one or more labels in your `Runner` or `RunnerDeployment` spec.

```yaml
# runnerdeployment.yaml
apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerDeployment
metadata:
  name: custom-runner
spec:
  replicas: 1
  template:
    spec:
      repository: actions-runner-controller/actions-runner-controller
      labels:
        - custom-runner
```

Once this spec is applied, you can observe the labels for your runner from the repository or organization in the GitHub settings page for the repository or organization. You can now select a specific runner from your workflow by using the label in `runs-on`:

```yaml
jobs:
  release:
    runs-on: custom-runner
```

Note that if you specify `self-hosted` in your workflow, then this will run your job on _any_ self-hosted runner, regardless of the labels that they have.

### Runner Groups

Runner groups can be used to limit which repositories are able to use the GitHub Runner at an organization level. Runner groups have to be [created in GitHub first](https://docs.github.com/en/actions/hosting-your-own-runners/managing-access-to-self-hosted-runners-using-groups) before they can be referenced.

To add the runner to the group `NewGroup`, specify the group in your `Runner` or `RunnerDeployment` spec.

```yaml
# runnerdeployment.yaml
apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerDeployment
metadata:
  name: custom-runner
spec:
  replicas: 1
  template:
    spec:
      group: NewGroup
```

### Using IRSA (IAM Roles for Service Accounts) in EKS

> This feature requires controller version => [v0.15.0](https://github.com/actions-runner-controller/actions-runner-controller/releases/tag/v0.15.0)

As similar as for regular pods and deployments, you firstly need an existing service account with the IAM role associated.
Create one using e.g. `eksctl`. You can refer to [the EKS documentation](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) for more details.

Once you set up the service account, all you need is to add `serviceAccountName` and `fsGroup` to any pods that uses the IAM-role enabled service account.

For `RunnerDeployment`, you can set those two fields under the runner spec at `RunnerDeployment.Spec.Template`:

```yaml
apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerDeployment
metadata:
  name: example-runnerdeploy
spec:
  template:
    spec:
      repository: USER/REO
      serviceAccountName: my-service-account
      securityContext:
        fsGroup: 1000
```


### Use with Istio

Istio 1.7.0 or greater has `holdApplicationUntilProxyStarts` added in https://github.com/istio/istio/pull/24737, which enables you to delay the `runner` container startup until the injected `istio-proxy` container finish starting. Try using it if you need to use Istio. Otherwise the runner is unlikely to work, because it fails to call any GitHub API to register itself due to `istio-proxy` being not up and running yet.

Note that there's no official Istio integration in actions-runner-controller. It should work, but it isn't covered by our acceptance test (a contribution to resolve this is welcomed). In addition to that, none of the actions-runner-controller maintainers use Istio daily. If you need more information, or have any issues using it, refer to the following links:

- https://github.com/actions-runner-controller/actions-runner-controller/issues/591
- https://github.com/actions-runner-controller/actions-runner-controller/pull/592
- https://github.com/istio/istio/issues/11130

### Stateful Runners

> This feature requires controller version => [v0.20.0](https://github.com/actions-runner-controller/actions-runner-controller/releases/tag/v0.20.0)

`actions-runner-controller` supports `RunnerSet` API that let you deploy stateful runners. A stateful runner is designed to be able to store some data persists across GitHub Actions workflow and job runs. You might find it useful, for example, to speed up your docker builds by persisting the docker layer cache.

A basic `RunnerSet` would look like this:

```yaml
apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerSet
metadata:
  name: example
spec:
  ephemeral: false
  replicas: 2
  repository: mumoshu/actions-runner-controller-ci
  # Other mandatory fields from StatefulSet
  selector:
    matchLabels:
      app: example
  serviceName: example
  template:
    metadata:
      labels:
        app: example
```

As it is based on `StatefulSet`, `selector` and `template.medatada.labels` needs to be defined and have the exact same set of labels. `serviceName` must be set to some non-empty string as it is also required by `StatefulSet`.

Runner-related fields like `ephemeral`, `repository`, `organization`, `enterprise`, and so on should be written directly under `spec`.

Fields like `volumeClaimTemplates` that originates from `StatefulSet` should also be written directly under `spec`.

Pod-related fields like security contexts and volumes are written under `spec.template.spec` like `StatefulSet`.

Similarly, container-related fields like resource requests and limits, container image names and tags, security context, and so on are written under `spec.template.spec.containers`. There are two reserved container `name`, `runner` and `docker`. The former is for the container that runs [actions runner](https://github.com/actions/runner) and the latter is for the container that runs a dockerd.

For a more complex example, see the below:

```yaml
apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerSet
metadata:
  name: example
spec:
  ephemeral: false
  replicas: 2
  repository: mumoshu/actions-runner-controller-ci
  dockerdWithinRunnerContainer: true
  template:
    spec:
      securityContext:
        #All level/role/type/user values will vary based on your SELinux policies.
        #See https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux_atomic_host/7/html/container_security_guide/docker_selinux_security_policy for information about SELinux with containers
        seLinuxOptions: 
          level: "s0"
          role: "system_r"
          type: "super_t"
          user: "system_u"
      containers:
      - name: runner
        env: []
        resources:
          limits:
            cpu: "4.0"
            memory: "8Gi"
          requests:
            cpu: "2.0"
            memory: "4Gi"
      - name: docker
        resources:
          limits:
            cpu: "4.0"
            memory: "8Gi"
          requests:
            cpu: "2.0"
            memory: "4Gi"
```

You can also read the design and usage documentation written in the original pull request that introduced `RunnerSet` for more information.

https://github.com/actions-runner-controller/actions-runner-controller/pull/629

Under the hood, `RunnerSet` relies on Kubernetes's `StatefulSet` and Mutating Webhook. A statefulset is used to create a number of pods that has stable names and dynamically provisioned persistent volumes, so that each statefulset-managed pod gets the same persistent volume even after restarting. A mutating webhook is used to dynamically inject a runner's "registration token" which is used to call GitHub's "Create Runner" API.

We envision that `RunnerSet` will eventually replace `RunnerDeployment`, as `RunnerSet` provides a more standard API that is easy to learn and use because it is based on `StatefulSet`, and it has a support for `volumeClaimTemplates` which is crucial to manage dynamically provisioned persistent volumes.

**Limitations**

* For autoscaling the `RunnerSet` kind only supports pull driven scaling or the `workflow_job` event for webhook driven scaling.
* For autoscaling the `RunnerSet` kind doesn't support the [registration-only runner](#autoscaling-tofrom-0), these are deprecated however and to be [removed](https://github.com/actions-runner-controller/actions-runner-controller/issues/859)
* A known down-side of relying on `StatefulSet` is that it misses a support for `maxUnavailable`. A `StatefulSet` basically works like `maxUnavailable: 1` in `Deployment`, which means that it can take down only one pod concurrently while doing a rolling-update of pods. Kubernetes 1.22 doesn't support customizing it yet so probably it takes more releases to arrive. See https://github.com/kubernetes/kubernetes/issues/68397 for more information.

### Ephemeral Runners

Both `RunnerDeployment` and `RunnerSet` has ability to configure `ephemeral: true` in the spec.

When it is configured, it passes a `--once` flag to every runner.

`--once` is an experimental `actions/runner` feature that instructs the runner to stop after the first job run. It has a known race condition issue that means the runner may fetch a job even when it's being terminated. If a runner fetched a job while terminating, the job is very likely to fail because the terminating runner doesn't wait for the job to complete. This is tracked in issue [#466](https://github.com/actions-runner-controller/actions-runner-controller/issues/466).

Since the implementation of the `--once` flag GitHub have implemented the `--ephemeral` flag which has no known race conditions and is much more supported by GitHub, this is the prefered flag for ephemeral runners. To have your `RunnerDeployment` and `RunnerSet` kinds use this new flag instead of the `--once` flag set `RUNNER_FEATURE_FLAG_EPHEMERAL` to `"true"`. For example, a `RunnerSet` configured to use the new flag looks like:

```yaml
kind: RunnerSet
metadata:
  name: example-runnerset
spec:
  # ...
  template:
    metadata:
      labels:
        app: example-runnerset
    spec:
      containers:
      - name: runner
        imagePullPolicy: IfNotPresent
        env:
        - name: RUNNER_FEATURE_FLAG_EPHEMERAL
          value: "true"
```

You should configure all your ephemeral runners to use the new flag unless you have a reason for needing to use the old flag.

Once able, `actions-runner-controller` will make `--ephemeral` the default option for `ephemeral: true` runners and potentially remove `--once` entirely. It is likely that in the future the `--once` flag will be officially deprecated by GitHub and subsquently removed in `actions/runner`.

### Software Installed in the Runner Image

**Cloud Tooling**<br />
The project supports being deployed on the various cloud Kubernetes platforms (e.g. EKS), it does not however aim to go beyond that. No cloud specific tooling is bundled in the base runner, this is an active decision to keep the overhead of maintaining the solution manageable.

**Bundled Software**<br />
The GitHub hosted runners include a large amount of pre-installed software packages. GitHub maintain a list in README files at <https://github.com/actions/virtual-environments/tree/main/images/linux>

This solution maintains a few runner images with `latest` aligning with GitHub's Ubuntu version, these images do not contain all of the software installed on the GitHub runners. The images contain the following subset of packages from the GitHub runners:

- Basic CLI packages
- git
- docker
- build-essentials

The virtual environments from GitHub contain a lot more software packages (different versions of Java, Node.js, Golang, .NET, etc) which are not provided in the runner image. Most of these have dedicated setup actions which allow the tools to be installed on-demand in a workflow, for example: `actions/setup-java` or `actions/setup-node`

If there is a need to include packages in the runner image for which there is no setup action, then this can be achieved by building a custom container image for the runner. The easiest way is to start with the `summerwind/actions-runner` image and installing the extra dependencies directly in the docker image:

```shell
FROM summerwind/actions-runner:latest

RUN sudo apt update -y \
  && sudo apt install YOUR_PACKAGE
  && sudo rm -rf /var/lib/apt/lists/*
```

You can then configure the runner to use a custom docker image by configuring the `image` field of a `Runner` or `RunnerDeployment`:

```yaml
apiVersion: actions.summerwind.dev/v1alpha1
kind: Runner
metadata:
  name: custom-runner
spec:
  repository: actions-runner-controller/actions-runner-controller
  image: YOUR_CUSTOM_DOCKER_IMAGE
```

### Using without cert-manager

Assuming you are installing in the default namespace, ensure your certificate has SANs:

* `webhook-service.actions-runner-system.svc`
* `webhook-service.actions-runner-system.svc.cluster.local`

It is possible to use a self-signed certificate by following a guide like
[this one](https://mariadb.com/docs/security/encryption/in-transit/create-self-signed-certificates-keys-openssl/)
using `openssl`.

Install your certificate as a TLS secret:

```shell
$ kubectl create secret tls webhook-server-cert \
  -n actions-runner-system \
  --cert=path/to/cert/file \
  --key=path/to/key/file
```

Set the Helm chart values as follows:

```shell
$ CA_BUNDLE=$(cat path/to/ca.pem | base64)
$ helm --upgrade install actions-runner-controller/actions-runner-controller \
  certManagerEnabled=false \
  admissionWebHooks.caBundle=${CA_BUNDLE}
```

### Common Errors

#### invalid header field value

```json
2020-11-12T22:17:30.693Z	ERROR	controller-runtime.controller	Reconciler error	
{
  "controller": "runner",
  "request": "actions-runner-system/runner-deployment-dk7q8-dk5c9",
  "error": "failed to create registration token: Post \"https://api.github.com/orgs/$YOUR_ORG_HERE/actions/runners/registration-token\": net/http: invalid header field value \"Bearer $YOUR_TOKEN_HERE\\n\" for key Authorization"
}
```

**Solution**

Your base64'ed PAT token has a new line at the end, it needs to be created without a `\n` added, either:
* `echo -n $TOKEN | base64`
* Create the secret as described in the docs using the shell and documented flags

#### Runner coming up before network available

If you're running your action runners on a service mesh like Istio, you might
have problems with runner configuration accompanied by logs like:

```
....
runner Starting Runner listener with startup type: service
runner Started listener process
runner An error occurred: Not configured
runner Runner listener exited with error code 2
runner Runner listener exit with retryable error, re-launch runner in 5 seconds.
....
```

This is because the `istio-proxy` has not completed configuring itself when the
configuration script tries to communicate with the network.

**Solution**<br />

> Added originally to help users with older istio instances.
> Newer Istio instances can use Istio's `holdApplicationUntilProxyStarts` attribute ([istio/istio#11130](https://github.com/istio/istio/issues/11130)) to avoid having to delay starting up the runner.
> Please read the discussion in [#592](https://github.com/actions-runner-controller/actions-runner-controller/pull/592) for more information.

_Note: Prior to the runner version v2.279.0, the environment variable referenced below was called `STARTUP_DELAY`._

You can add a delay to the runner's entrypoint script by setting the `STARTUP_DELAY_IN_SECONDS` environment
variable for the runner pod. This will cause the script to sleep X seconds, this works with any runner kind.

*Example `RunnerDeployment` with a 2 second startup delay:*
```yaml
apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerDeployment
metadata:
  name: example-runnerdeployment-with-sleep
spec:
  template:
    spec:
      env:
        - name: STARTUP_DELAY_IN_SECONDS
          value: "2" # Remember! env var values must be strings.
```

# Contributing

For more details on contributing to the project (including requirements) please check out [Getting Started with Contributing](CONTRIBUTING.md).
