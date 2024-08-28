# Base Platform Baseline Repo


[TOC]

## Introduction
This repo is used to hold the baseline for base platform applications.
All the Base applications, including their CRDs, are stored in a helmfile format within the helmfile directory at the root
of the repo. Each update to the helmfile content causes the version in the metadata.yaml file to be stepped. This ensures all changes
can be mapped to a specific version of the Base Platform Baseline. Please see the [git web link](https://gerrit-gamma.gic.ericsson.se/gitweb?p=OSS%2Fcom.ericsson.oss.aeonic%2Foss-base-baseline.git;a=shortlog;h=refs%2Fheads%2Fmaster)
for an example of the tags attached to each of the new pushes to master.

The baseline repo is used within the Base Platform staging area and with the different PSO areas.

In the **Base Platform flows**, the new application helm chart will be added to the baseline and a snapshot created.
If this snapshot is successful in the base platform flow then a new version of the base platform baseline is released.

In the **PSO flows**, the newly released baseline version is used to retrieve all the base platform applications and
their associated CRDs. All the base applications will be executed in the PSO flow. This ensures that each PSO flow is
kept in sync and ensures that the version being tested in PSO have already gone through a test flow together.

Please note in the majority of the deployments in PSO for base applications there is still only one change per run if
the previous run for base was successful.

## New Base Application Initial delivery
Currently, all the base applications are delivered individually to the project helmfiles.

When introducing a new base application initially, it necessitates a manual review of the project helmfile. This is
because adjustments are required at the helmfile level to incorporate the new application chart. An illustration of such
a review is available [here](https://gerrit-gamma.gic.ericsson.se/#/c/17192085/) for the introduction of SEF into the EO Helmfile.

Despite the introduction of the Base Platform Baseline, this procedure remains unchanged. New base applications can
still undergo review for inclusion in the helmfile via the review channel.

Upon successful testing and delivery to the respective project helmfiles through the Product staging flows, the new base
platform application should be integrated into the Base Platform Baseline repository. The new application details must
be added to the appropriate helmfiles within the base platform baseline repository. Refer to the existing `helmfile.yaml`
or `crd_helmfile.yaml` in the directory
[helmfile/base-platfrom-baseline](https://gerrit-gamma.gic.ericsson.se/plugins/gitiles/OSS/com.ericsson.oss.aeonic/oss-base-baseline/+/refs/heads/master/helmfile/base-platform-baseline)
for guidance.

This inclusion allows for ongoing testing within the base platform flow.

The review initiated for this change in the base platform baseline repository can be directly merged into the master
branch after review. It will undergo testing in the subsequent iteration through the base platform flow following the
merge.

Future deliveries of this new application should be managed in the usual manner through the automated release flow
within the Base Platform flow.

## Delivering a Base Application that doesn't have an automated release flow set-up

This process is used for base applications that may be early in their development, where the automated release flow may
not be set-up. This flow execution should be the last resort. The automated release flow should be the default process.

  - Execute the following job, [BASE PLATFORM MANUAL release E2E flow](https://spinnaker.rnd.gic.ericsson.se/#/applications/base-platform-e2e-cicd/executions?pipeline=BASE-PLATFORM-MANUAL-release-E2E-flow)
      - The following parameters need to be updated
        - CHART_NAME: "BASE CHART NAME"
        - CHART_REPO: "BASE CHART REPO"
        - CHART_VERSION: "BASE CHART VERSION"
        - GERRIT_CHANGE_URL: "GERRIT URL ASSOCIATED TO THE BASE CHART"
        - GIT_COMMIT_SUMMARY: "GERRIT MESSAGE ASSOCIATED WITH THE REVIEW"
        - DEPLOY_EO: <Set to true or false>
        - DEPLOY_EIAP: <Set to true or false>
      - During the execution it will generate and test a new base platform baseline and deliver to the PSO areas.

## Trigger a Base Platform delivery at the PSO Staging flow Manually
This process can be used if there was an issue between the Base platform staging flow and the PSO flow. Where the new
Base Platform Baseline version has been created but failed to execute the PSO flow for whatever reason.

**_NOTE:_** There is no need to execute this process unless the delivery is a high priority to get through PSO. As on
the next delivery to the Base Platform to PSO this update will be included also.

- Kick off the PSO flow manually using the Baseline version required
    - The following parameters will be required to execute the delivery:
      - See [BLT](https://blt-staging.ews.gic.ericsson.se/) for the Baseline version and chart details changed in that
      version,
        - BASE_PLATFORM_BASELINE_NAME: "base-platform-baseline"
        - BASE_PLATFORM_BASELINE_REPO: "https://arm.seli.gic.ericsson.se/artifactory/proj-eric-oss-drop-helm"
        - BASE_PLATFORM_BASELINE_VERSION: "BASE PLATFORM VERSION (See BLT)"
        - CHART_NAME: "CHART NAME CHANGED IN THE BASE BASELINE VERSION ABOVE"
        - CHART_REPO: "CHART REPO CHANGED IN THE BASE BASELINE VERSION ABOVE"
        - CHART_VERSION: "CHART VERSION CHANGED IN THE BASE BASELINE VERSION ABOVE"

## Differentiate applications between projects
There maybe instances where a certain project does not need a certain base application. This can be managed by the use
of labels within the release sections of the helmfiles.

See below for an example, it shows that eric-data-wide-column-database-cd-crd is only applicable for the EIAP project,
"labels.project.eric-eiae-helmfile". Any other project will not pick up this application information if set correctly
in the
[Set_Or_Get_Base_Platform_Baseline_App_Versions](https://gerrit-gamma.gic.ericsson.se/plugins/gitiles/OSS/com.ericsson.oss.aeonic/oss-integration-ci/+/refs/heads/master/docs/files/Set_Or_Get_Base_Platform_Baseline_App_Versions.md) 
jenkins file
  ```
  - name: eric-data-wide-column-database-cd-crd
    namespace: {{ .Values | get "helmfile.crd.namespace" "eric-crd-ns" }}
    chart: {{ .Values | get "repository" "adp-gs-all" }}/eric-data-wide-column-database-cd-crd
    labels:
        project: eric-eiae-helmfile
    version: 1.23.0+30
    values:
      - "./values-templates/global-values.yaml.gotmpl"
      - "./values-templates/release-site-values.yaml.gotmpl"
  ```

## Base Platform affecting PSO Staging flow

Follow this process if an application delivered to the Base Platform baseline breaks the PSO flow.
<br>This process should be followed if the issue is not observed in other PSO areas. If all PSO areas are broken please
see the [Rollback](#Rollback) section.

1. Create a support ticket on the base application that broke the pipeline.
2. The base platform baseline is made up of many applications which will include the offending application causing this
issue. All deliverables from base platform should be blocked from delivering into PSO until a resolution is found.
3. Once the fix is available, unblock the Base Platform Baseline delivery for the PSO flow.
<br>**_NOTE:_** On the next delivery of a base platform application into PSO, the project helmfile will be brought back
up to date with all the base deliveries that happened during the PSO block.

If a fix is not available in a timely manner, an Emergency Package(EP) may be needed. See section below on
[Emergency Package (EP) for specific PSO flow](#Emergency-Package-EP_for-specific-PSO-flow)

## Rollback
A Rollback may be necessary when all flows are seeing a similar issue with a base application.
The decision to rollback needs to be a joint decision between all the affected areas. Ticketmaster will coordinate this
decision between the areas if needed.

There are two options to perform a rollback,
- Option 1: Rollback the PSO area to a known good baseline
  - Kick off the PSO flow manually using the Baseline version required
    - The following parameters will be required to be updated accordingly.
      - See [BLT](https://blt-staging.ews.gic.ericsson.se/) for the Baseline version and chart details changed in that
      version,
        - BASE_PLATFORM_BASELINE_NAME: "base-platform-baseline"
        - BASE_PLATFORM_BASELINE_REPO: "https://arm.seli.gic.ericsson.se/artifactory/proj-eric-oss-drop-helm"
        - BASE_PLATFORM_BASELINE_VERSION: "VERSION TO REVERT TO"
        - CHART_NAME: "CHART NAME CHANGED IN THE BASE BASELINE VERSION ABOVE"
        - CHART_REPO: "CHART REPO CHANGED IN THE BASE BASELINE VERSION ABOVE"
        - CHART_VERSION: "CHART VERSION CHANGED IN THE BASE BASELINE VERSION ABOVE"
        - ALLOW_BASE_BASELINE_DOWNGRADE: true
  - Block all deliveries from Base Platform at the PSO level until a fix is available.

- Option 2: Generate a new version of the Base Platform Baseline through Base Platform Flow
  - This is only used when you need to completely rollback a base version out of all PSO areas.
  - Please register a ticket on Ticketmaster to execute this process. Register a ticket using JIRA template
  [IDUN-4091](https://jira-oss.seli.wh.rnd.internal.ericsson.com/browse/IDUN-4091)

## Emergency Package (EP) for specific PSO flow
As a last resort it may be necessary for an emergency package to be created. This is where an application chart within the base
platform Baseline is faulty for a certain PSO flow.

This emergency package is created by the Ticketmaster team. An EP request should be sent to Ticketmaster using the
following JIRA template [IDUN-4091](https://jira-oss.seli.wh.rnd.internal.ericsson.com/browse/IDUN-4091)

**_NOTE:_** EPs are executed manually, if subsequent base platform applications are required in the PSO area, a new EP
would need to be requested and executed.
