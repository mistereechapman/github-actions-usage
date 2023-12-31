[![Security Rating](https://sonarcloud.io/api/project_badges/measure?project=fergusmacd_github-action-usage&metric=security_rating)](https://sonarcloud.io/summary/new_code?id=fergusmacd_github-action-usage)

# GitHub Actions Usage Audit

This GitHub Action can:

- fails when remaining minutes drop below a defined number
- print out monthly action minutes budget
- print out action usage per organization repo
- print out action usage per organization repo and workflow
- show totals for both of the above
- print out the number of remaining days in the billing cycle
- be run locally with Docker or python
- write files locally for collecting(CSV+JSON)

## What is the Reason for this Action?

Original story below - I just wanted to make this better. 

This action came about because a project I was working on kept running out of free minutes very quickly. As an admin I
was asking teams to reduce usage, but they had no idea which repos and workflows were clocking up the minutes. Worse,
the billing user was away for a day, the free minutes ran out, and we had no buffer set up. Development and deployment
came to a virtual standstill. So I started investigating ways to make usage more transparent.

With the use of GitHub Actions (GHA) in build and deploy pipelines, usage adda up very quickly. Once the monthly limit
has been reached, as we found out, the workflows will just stop running thereby causing huge disruption in the build and
delivery pipeline. If the billing user is on holiday, this is pretty disastrous. Better to be forewarned and so extra
credits can be added by the billing owner.

However, the usage minutes total is hidden away in the admin section and so repo owners cannot even see GHA usage. Even
if they could, the usage CSV that GitHub sends out contains too much information making it hard to isolate heavy usage
workflows and repos.

Furthermore, MacOS usage is charged at 10x, Windows 2x the rate of Ubuntu machines. This means that a 7 second action
runtime will be consume 1 minute of the allowance on Ubuntu, 10 minutes on MacOS, and 2 minutes on Windows. MacOS builds
can take 20-30 minutes and the free minutes soon dry up. It turned out, this was the cause of our high usage. Top tip:
don't build MacOS machines in GitHub.

## What the Action Does

So I wrote this action to address the problems above, in the following way:

- fails the workflow if the minutes remaining drops below 100, or a user defined value meaning a notification will be
  sent out to watchers
- show remaining minutes left in billing period
- give clear visibility of GitHub Action usage to all users
- show total usage per repo
- show total usage per repo and workflow
- show usage by machine type, i.e. Ubuntu, MacOS and Windows

To do this, the GHA prints out two tables:

- total usage per repo
- usage per repo and workflow

in the prettyprint formatted ASCII tables like this:

```
+--------------------------------+--------+-------+---------+
| Repo Name                      | Ubuntu | MacOS | Windows |
+------------------------------- +--------+-------+---------+
| aws-infra                      |   0    |   0   |    0    |
| cicd-images                    |   12   |   0   |    0    |
| terraform-github-repository    |   39   |   0   |    0    |
| ---------                      |  ----  |  ---- |   ----  |
| Usage Minutes 2022-06-13 13:59 |   51   |   0   |    0    |
| ---------                      |  ----  |  ---- |   ----  |
| Stats From GitHub              |        |       |         |
| Monthly Allowance: 2000        |        |       |         |
| Usage Minutes: 51              |   51   |   0   |    0    |
| Remaining Minutes: 1949        |        |       |         |
| Alarm Triggered at: 150        |        |       |         |
| Paid Minutes: 0                |        |       |         |
| Days Left in Cycle: 13         |        |       |         |
+--------------------------------+--------+-------+---------+

+--------------------------------+---------------------+--------+-------+---------+
| Repo Name                      | Workflow            | Ubuntu | MacOS | Windows |
+--------------------------------+---------------------+--------+-------+---------+
| aws-infra                      | automerge.yml       |   0    |   0   |    0    |
|                                | close-stale-prs.yml |   0    |   0   |    0    |
|                                | enforce-labels.yml  |   0    |   0   |    0    |
|                                | labeler.yml         |   0    |   0   |    0    |
|                                | release.yml         |   0    |   0   |    0    |
|                                | setup-terraform.yml |   0    |   0   |    0    |
| --------                       | --------            | -----  | ----- |  -----  |
| --------                       | --------            | -----  | ----- |  -----  |
| github-audit                   | automerge.yml       |   0    |   0   |    0    |
|                                | close-stale-prs.yml |   15   |   0   |    0    |
|                                | enforce-labels.yml  |   0    |   0   |    0    |
|                                | labeler.yml         |   0    |   0   |    0    |
|                                | release.yml         |   0    |   0   |    0    |
|                                | setup-terraform.yml |   0    |   0   |    0    |
| --------                       | --------            | -----  | ----- |  -----  |
| terraform-github-repository    | No workflows        |        |       |         |
| --------                       | --------            | -----  | ----- |  -----  |
| Usage Minutes 2022-06-13 13:59 |                     |   15   |   0   |    0    |
| --------                       | --------            | -----  | ----- |  -----  |
| Stats From GitHub              |                     |        |       |         |
| Monthly Allowance: 2000        |                     |        |       |         |
| Usage Minutes: 15              |                     |   15   |   0   |    0    |
| Remaining Minutes: 1985        |                     |        |       |         |
| Alarm Triggered at: 150        |                     |        |       |         |
| Paid Minutes: 0                |                     |        |       |         |
| Days Left in Cycle: 13         |                     |        |       |         |
+--------------------------------+---------------------+--------+-------+---------+
```

It also writes the files to disc for collection by external services

## How Does it Work?

The action calls GitHub REST API endpoints to get the required information, and then prettyprint for formatting.

To get the number of repos, it
calls [GitHub Organisation API](https://docs.github.com/en/rest/orgs/orgs#get-an-organization). For information on each
repo, it
calls [GitHub List Organisational Repos API](https://docs.github.com/en/rest/repos/repos#list-organization-repositories)
. For repository workflows, it
calls [GitHub List Repository Workflow API](https://docs.github.com/en/rest/actions/workflows#list-repository-workflows)
. For workflow usage, it
calls [GitHub Get Workflow Usage API](https://docs.github.com/en/rest/actions/workflows#get-workflow-usage).

For days left in the billing cycle , it
calls [GitHub Get Actions billing for an organization API](https://docs.github.com/en/rest/billing#get-shared-storage-billing-for-an-organization)
.

Finally for monthly allowance, paid minutes and what GitHub think has been used it
calls [GitHub Get shared storage billing for an organization API](https://docs.github.com/en/rest/billing#get-github-actions-billing-for-an-organization)

## Prerequisites to Run as an GH Action

- an organisation or repo secret called `ORGANISATION` with the value of your organisation
- a secret called `GITHUBAPIKEY` with the value being a personal access token (PAT) with scope `read:org` for reading
  public repos and `repo:full` for reading private repos.

## Usage

### As a GitHub Action

Create a file called `gha-audit.yml` in your `workflows` directory, paste the following as the contents and you are good
to
go. [GHA best practices](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions#using-third-party-actions)
recommend using a commit SHA, rather than a version. The example below runs on a schedule at 3AM every day. This way
when the remaining allowance drops below the threshold (100 or user defined) a notification will be triggered.

```
name: GHA Usage Audit
on:
  schedule:
    - cron: "0 3 * * *" # Runs at 03:00 AM (UTC) every day
jobs:
  gha-usage-minutes-report:
    runs-on: ubuntu-latest
    steps:
      - name: GitHub Actions Usage Audit
        uses: mistereechapman/github-actions-usage@v1.0.0
        # pass user input as arguments
        with:
          organisation: ${{secrets.ORGANISATION}}
          gitHubAPIKey: ${{secrets.GITHUBAPIKEY}} # default token in GitHub Workflow
          loglevel: error # not required, change to debug if misbehaving
          raisealarmremainingminutes: 100 # not required, defaults to 100
          skipReposWithoutUsage: false
```

### Running Locally

The docker file and python script can both be run locally in the following ways.

#### Running with Python

For python, from the python directory:

```shell
pip install -r requirements.txt
# default is warning, see the action.yaml for further details
# GHA environment variables prepend INPUT_ to values passed in
export INPUT_LOGLEVEL=debug|info|warning|error
export INPUT_ORGANISATION="myorg"
export INPUT_GITHUBAPIKEY="***"
export INPUT_RAISEALARMREMAININGMINUTES="150"
export INPUT_SKIPREPOSWITHOUTUSAGE="false"
export 
# from python directory you can run
python main.py
```

#### Running with Docker

For Docker, run from the root directory:

```shell
# from root directory
docker build -t gha-billable-usage .
# GHA environment variables prepend INPUT_ to values passed in
export INPUT_LOGLEVEL=debug|info|warning|error
export INPUT_ORGANISATION="myorg"
export INPUT_GITHUBAPIKEY="***"
export INPUT_RAISEALARMREMAININGMINUTES="150"
export INPUT_SKIPREPOSWITHOUTUSAGE="false"
docker run -v $PWD:/app/results -e INPUT_RAISEALARMREMAININGMINUTES=${INPUT_RAISEALARMREMAININGMINUTES} -e INPUT_LOGLEVEL=${INPUT_LOGLEVEL} -e INPUT_ORGANISATION=${INPUT_ORGANISATION} -e INPUT_GITHUBAPIKEY=${INPUT_GITHUBAPIKEY} -e INPUT_SKIPREPOSWITHOUTUSAGE=${INPUT_SKIPREPOSWITHOUTUSAGE} -it gha-billable-usage
```

## Common Errors

When problems happen, the best thing to do is set the log level to `debug` like this locally:

```shell
export LOGLEVEL="debug"
```

Or change the loglevel input in the GHA.

The following one happens when running locally and the `INPUT_GITHUBAPIKEY` environment variable has not been exported:

```shell
python3 main.py
Traceback (most recent call last):
  File "/github-action-usage/python/main.py", line 5, in <module>
    from ghaworkflows import *
  File "/github-action-usage/python/ghaworkflows.py", line 9, in <module>
    github_api_key = getgithubapikey()
  File "/github-action-usage/python/common.py", line 24, in getgithubapikey
    return os.environ['INPUT_GITHUBAPIKEY']
  File "/usr/local/Cellar/python@3.9/3.9.12/Frameworks/Python.framework/Versions/3.9/lib/python3.9/os.py", line 679, in __getitem__
    raise KeyError(key) from None
KeyError: 'INPUT_GITHUBAPIKEY'

```

This error happens when the PAT has expired or does not have sufficient permissions:

```shell
python3 main.py                                                     
Traceback (most recent call last):
  File "/github-action-usage/python/main.py", line 92, in <module>
    main()
  File "/github-action-usage/python/main.py", line 30, in main
    repoNames = getreposfromorganisation(org)
  File "/github-action-usage/python/ghorg.py", line 27, in getreposfromorganisation
    totalPrivateRepos = json_data["total_private_repos"]
KeyError: 'total_private_repos'
```

## Relevant Links

[Billing documentation](https://docs.github.com/en/billing/managing-billing-for-github-actions/about-billing-for-github-actions#calculating-minute-and-storage-spending)

The following APIs are used:

- [GitHub Organisation API](https://docs.github.com/en/rest/orgs/orgs#get-an-organization) - to get the number of repos
- [GitHub List Organisational Repos API](https://docs.github.com/en/rest/repos/repos#list-organization-repositories) -
  to get the repo information
- [GitHub List Repository Workflow API](https://docs.github.com/en/rest/actions/workflows#list-repository-workflows) -
  to get the repository workflows
- [GitHub Get Workflow Usage API](https://docs.github.com/en/rest/actions/workflows#get-workflow-usage) - for workflow
  usage
- [GitHub Get shared storage billing for an organization API](https://docs.github.com/en/rest/billing#get-shared-storage-billing-for-an-organization)
    - for days left in billing cycle

There are plenty of tutorials on prettyprint, I used this one:

- [Zetcode](https://zetcode.com/python/prettytable/)

## Road Map Items

- alerts when limits are closed to being reached
- export to excel and upload to packages
- sorting by different criteria, e.g. tags or ownership
- test coverage
- test scripts
- colouring console
- any requests?
