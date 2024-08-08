# JFrog Log Analytics Changelog

All changes to the log analytics integration will be documented in this file.

## [1.0.10] - August 8, 2024

* Fix metrics configuration due to deprication of `artifactory.openMetrics` as part of Artifactory 7.87.x charts and renaming it to `artifactory.metrics`

## [1.0.9] - July 22, 2024

* FluentD sidecar version bumped to 4.5, to upgrade base image to bitnami/fluentd 1.17.0
* Fixing metrics documentation

## [1.0.8] - June 7, 2024

* [BREAKING] Adding deprecation notice for partnership-pts-observability.jfrog.io docker registry
* FluentD sidecar version bumped to 4.3, to upgrade base image to bitnami/fluentd 1.16.5
* Update helm charts for FluentD sidecar to match recent init containers charts changes in JFrog's official charts

## [1.0.7] - May 8, 2024

* Fix issue where loki helm values file not compatible with loki 3.x

## [1.0.6] - April 25, 2024

* Fix Artifactory Grafana dashbaord to count correctly number of accessed docker images and repos

## [1.0.5] - April 18, 2024

* Fix fluentd regex to fetch correctly docker image and repo names

## [1.0.4] - April 09, 2024

* Fix access log regex pattern to match log structure

## [1.0.3] - April 05, 2024

* Adding multi-arch support (amd64, arm64) for the sidecar docker image to allow also deployment in ARM environments
* Using loki charts instead of the deprecated loki-stack charts
* Updating Readme installation instructions

## [1.0.2] - Jan 10, 2024

* Upgrading fluentd image to the latest image from releases.jfrog.io (Fluentd version 1.16.3)

## [1.0.1] - Nov 14, 2023

* Upgrading fluentd image to the latest image from releases.jfrog.io (Fluentd version 1.15.2)
* Removing sed substitutions in helm charts to eliminate human errors in loki URL

## [1.0.0] - Dec 12, 2022

* Prometheus Solution Redesigned
* Addition of Loki Component for Log Data Ingestion
* Enhanced Prometheus for Metrics Data Ingestion

## [0.15.0] - Oct 14, 2022

* Deprecation notice

## [0.10.0] - Feb 14, 2022

* Added call home configuration for the artifactory fluent configuration.

## [0.9.0] - Dec 31, 2020

* Repo layout fixes
* Support for Helm
* Readme updates

## [0.8.0] - Nov 30, 2020

* Bug fixes

## [0.7.2] - Oct 29, 2020

* Updated docker chart queries to monitor [dockerhub rate limiting](https://jfrog.com/blog/get-around-docker-download-limits-jfrog-artifactory/).

## [0.7.1] - Oct 28, 2020

* Updated fluentd config, added new chart widgets and an alert to monitor [dockerhub rate limiting](https://jfrog.com/blog/get-around-docker-download-limits-jfrog-artifactory/).

## [0.7.0] - Oct 20, 2020

* Fixing issue with ip_address in access logs having space and . at the end

## [0.6.1] - Sept 25, 2020

* Updated Prometheus and Grafana installation to use the Prometheus community kube-prometheus-stack. Added drill down options to charts.

## [0.6.0] - Sept 25, 2020

* [BREAKING] Prometheus fluentd configs updated to use JF_PRODUCT_DATA_INTERNAL env.

## [0.5.1] - Sept 9, 2020

* Prometheus repo now submodule of parent log-analytics

## [0.5.0] - Sept 8, 2020

* [No Prometheus] Adding JFrog Pipelines fluent configuration files to capture logs

## [0.4.0] - Sept 4, 2020

* [No Prometheus] Adding JFrog Mission Control fluent configuration files to capture logs

## [0.3.0] - Aug 26, 2020

* [No Prometheus] Adding JFrog Distribution fluent configuration files to capture logs

## [0.2.0] - Aug 24, 2020

* Splunk updates to launch new version of Splunkbase app v1.1.0

## [0.1.1] - June 1, 2020

* Removing the need for user to specify splunk host , user, and token twice
* Fixing issue with regex on the audit security log
* Fixed issue with the repo and image when not docker api url

## [0.1.0] - May 12, 2020

* Initial release of Jfrog Logs Analytic integration
