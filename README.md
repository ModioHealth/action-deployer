# deploy action
This action consolidates deployment logic into a single (easy-to-use) action. The following functionality is supported:
- Tokenize any helm chart with based on deployment data and deploy via ArgoCD (available tokens documented below)
- Automatically remove ArgoCD deployment on branch delete (useful for removing preview environments)
- Automatically open a draft GitHub pull request on branch creation
- Automatically link to all deployed services as GitHub pull request comment
- Add deployment information to GitHub using [Deployment API](https://developer.github.com/v3/guides/delivering-deployments/)
- [Annotate](https://grafana.com/docs/grafana/latest/dashboards/annotations/) deployment information in Grafana
- Add [deployment information](https://developer.atlassian.com/cloud/jira/software/rest/api-group-deployments/#api-group-deployments) to JIRA
- Create [Sentry release](https://docs.sentry.io/cli/releases/) for deployment

This action is designed to handle the following [event types](https://docs.github.com/en/actions/reference/events-that-trigger-workflows):
- [deployment](https://docs.github.com/en/actions/reference/events-that-trigger-workflows#deployment)
- [deployment_status](https://docs.github.com/en/actions/reference/events-that-trigger-workflows#deployment_status)
- [delete](https://docs.github.com/en/actions/reference/events-that-trigger-workflows#delete)
- [check_suite](https://docs.github.com/en/actions/reference/events-that-trigger-workflows#check_suite)

## quickstart
This action requires a valid helm values file to work. You'll need to ensure that the file specified at `helmChartValuesFilePath` exists in order to use this action. To get started with this action (with no modules enabled), add this to your github actions file:


```yaml
- id: deploy
  uses: modiohealth/action-deployer@next
  with:
    appName: api
    dockerRepository: 999999999999.dkr.ecr.us-west-2.amazonaws.com/demo-app
    domain: example.com
    githubToken: someTokenHere
    helmChartRepoUrl: http://example.io/helm-charts
    helmChartName: some-random-chart
    helmChartVersion: '1.0.1'
    helmChartValuesFilePath: ./deploy/app.yml
    subdomain: api
```

_Note_: if you want to perform any deployments you will need to enable the argocd module (see configuration below):

_NOTE 2_: the value of `secrets.GITHUB_TOKEN` does not have the necessary permissions to perform some of the actions performed by this deployment. To remove this restriction you should generate a personal access token and use that instead. More information can be found [here](https://docs.github.com/en/actions/configuring-and-managing-workflows/authenticating-with-the-github_token#permissions-for-the-github_token).

## core configuration
The following core configuration should be set regardless of which modules are enabled for a deployment:

| Required | Parameter | Description | Default Value |
|----------|-----------|-------------|---------------|
|x| `appName` | Application name (defaults to repository name) | `null` |
| | `appVersion` | Application version (defaults to branch name) | `null` |
| | `deployPreviewsEnabled` | Enable deploy previews (defaults to true) | `true` |
|x| `dockerRepository` | Docker repository deploy from | `null` |
| | `domain` | Domain name to use for deployment (defaults to empty) | `null` |
| | `gateway` | Ingress Gateway to use for this service | `default-gateway-private` |
|x| `githubToken` | GitHub token used by action | `null` |
|x| `helmChartRepoUrl` | Default helm chart repository to use | `null` |
|x| `helmChartName` | Default helm chart to apply | `null` |
|x| `helmChartVersion` | Default helm chart version to use. See https://helm.sh/docs/topics/charts/ for information on versioning | `null` |
|x| `helmChartValuesFilePath` | YAML configuration file path for the chart that will be applied (relative path is fine). Will attempt to pull from this repo if file does not exist. | `null` |
| | `path` | Base path to use to access the app (defaults to empty string) | `''` |
| | `subdomain` | Base subdomain to access the app | `null` |


### helm values tokenization
Please note that a token replacement operation will be performed on the helm values file located at `helmChartValuesFilePath`. If any of the strings below exist in the yaml file they will be replaced with deployment specific values:

| Token | Description | Example Value |
|-------|-------------|---------------|
| `__APPNAME__` | Application name (automatically generated using `appName`) | `demo-app-api` |
| `__APPVERSION__` | Version of the application (automatically generated based on branch name) | `develop`, `preview--branch-name` |
| `__DOCKERREPO__` | Docker repository to use | `999999999999.dkr.ecr.us-west-2.amazonaws.com/demo-app` |
| `__DOCKERTAG__` | Docker tag to use | `develop-efb45b1` |
| `__DOMAIN__` | Base domain for this service (if applicable) | `example.com` |
| `__GATEWAY__` | Ingress Gateway for this service (if applicable) | `default-gateway-private` |
| `__GIT_SHA__` | Full Git SHA | `2b8045460ea7442cc81325ceadcd1bb2cb6a3b9c` |
| `__PATH__` | HTTP path for this service (if applicable). Comes from `path`. | `/some-path` |
| `__SUBDOMAIN__` | Subdomain for this service (if applicable) | `api`, `api--my-preview` |


## module specific configuration
The following additional modules can be enabled/configured depending on your preferences.

### argocd (highly recommended)
This module is used to deploy the generated helm chart using [ArgoCD](https://argoproj.github.io/argo-cd/).

| Parameter | Description | Default Value |
|-----------|-------------|---------------|
|`argoEnabled` | ArgoCD Enabled | `false` |
|`argoBaseUrl` | Base Argo url | `''` |
|`argoClientId` | ArgoCD Client ID | `''` |
|`argoClientSecret` | ArgoCD Client Secret | `''` |
|`argoAccessToken` | ArgoCD Access Token | `''` |
|`argoNamespace` | ArgoCD Namespace | `default` |
|`argoProject` | ArgoCD Project | `default` |
|`clusterUrl` | Kubernetes Cluster URL to deploy to | `''` |

### elastic apm
This module adds APM specific links for deploy preview environments.

| Parameter | Description | Default Value |
|-----------|-------------|---------------|
| `apmEnabled` | Elastic APM Enabled | `false` |
| `apmBaseUrl` | Elastic APM Base URL | `null` |

### github
This module adds pull request comments with deployment preview specific links. Note that you must enable other modules in order for them to be included in pull request comments.

| Parameter | Description | Default Value |
|-----------|-------------|---------------|
| `githubEnabled` | GitHub enabled (defaults to false) | `false` |

### grafana
This module adds the following [annotations](https://grafana.com/docs/grafana/latest/dashboards/annotations/) to grafana. The following tags are created when the following events occur:
- Service Deployed 
  - `deployment:created`
  - App Environment (e.g. `development`, `preview`, `production`)
  - App Name (e.g. `demo-app-api`)
  - App Version (e.g. `develop`, `some-branch-name`)
- Service Destroyed
  - `deployment:destroyed`
  - App Environment (e.g. `development`, `preview`, `production`)
  - App Name (e.g. `demo-app-api`)
  - App Version (e.g. `develop`, `some-branch-name`)

| Parameter | Description | Default Value |
|-----------|-------------|---------------|
| `grafanaEnabled` | Grafana Deployment Annotations Enabled | `false` |
| `grafanaApiKey` | Grafana API Key | `null` |
| `grafanaBaseUrl` | Grafana Base URL | `null` |
  
### jira
| Parameter | Description | Default Value |
|-----------|-------------|---------------|
| `jiraEnabled` | JIRA Deployment Enabled | `false` |
| `jiraClientId` | JIRA Client ID | `null` |
| `jiraClientSecret` | JIRA Client Secret | `null` |
| `jiraOrganization` | JIRA Organization ID | `null` |
| `jiraOrganizationProperty` | JIRA Organization Property (metadata added to deployments) | `null` |

### kibana
This module adds kibana specific links for deploy preview environments (useful for fetching application logs).

| Parameter | Description | Default Value |
|-----------|-------------|---------------|
| `kibanaEnabled` | Kibana Enabled | `false` |
| `kibanaBaseUrl` | Kibana Base URL | `null` |
| `cloudAccountId` | Application Cloud Account ID (e.g. AWS account ID) | `null` |
| `kibanaIndex` | Kibana index to use | `null` |

### sentry
This module creates a sentry release when a deployment is successfully released.

| Parameter | Description | Default Value |
|-----------|-------------|---------------|
| `sentryEnabled` | Sentry Releases Enabled | `false` |
| `sentryOrganization` | Sentry Organization | `null` |
| `sentryProjectId` | Sentry Project ID | `null` |
| `sentryProjectSlug` | Sentry Project Slug | `null` |
| `sentryToken` | Sentry API Token | `null` |

### vault
This module updates the associated GitHub pull request to include a link to create a vault secret for a deployment if it is missing.

| Parameter | Description | Default Value |
|-----------|-------------|---------------|
| `vaultEnabled` | Vault Enabled | `null` |
| `vaultBaseUrl` | Vault Base URL | `null` |
| `vaultKeyPrefix` | Vault Key Prefix | `''` |
| `vaultApiKey` | Vault API Key | `null` |

## contributing
The default version of this action is currently hardcoded to `v1`. If action parameter names are changed, removed, or required new parameters are added then the action version will need to be updated to prevent any dependencies from breaking. Basically, be mindful if making changes to `action.yml`.

See `.github/workflows` for build and publishing instructions.

_NOTE_: GitHub actions does not currently support private actions to be shared across projects. No sensitive information should be hardcoded in this action.

Some useful documetation for creating actions:
- [`action.yml` documentation](https://help.github.com/en/articles/metadata-syntax-for-github-actions)
- [GitHub webhook event types](https://help.github.com/en/actions/automating-your-workflow-with-github-actions/events-that-trigger-workflows#webhook-events)
- [TypeScript definitions for webhooks](https://github.com/octokit/webhooks.js)
