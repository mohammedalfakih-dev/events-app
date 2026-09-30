# My contribution to Loc Events

I'm Mohammed Alfakih, one of the data engineers on HackYourFuture Class 55, Group A's final project. Loc Events combines imported Ticketmaster events and admin-created events in a Netherlands event-discovery app.

This repository is my personal fork of the [team repository](https://github.com/HackYourFutureProjects/c55-final-project-group-A). It preserves the shared application and its history. My contribution focuses on ingestion, publishing, orchestration integration and pipeline monitoring.

## Ticketmaster ingestion

I implemented the Ticketmaster Discovery API integration with validation and pagination in [PR #8](https://github.com/HackYourFutureProjects/c55-final-project-group-A/pull/8).

The ingestion work built on the maintainer-provided operational template. That foundation already supplied cloud and local-run infrastructure; my contribution adapted the source integration to Ticketmaster events.

I later expanded coverage in [PR #97](https://github.com/HackYourFutureProjects/c55-final-project-group-A/pull/97) using five consecutive 30-day date windows, paginating each window and deduplicating by Ticketmaster event ID. The reason was the source's deep-paging limit: continuing to increase the page number within one large search did not solve the coverage problem.

The development result recorded in that PR is evidence for that run, not a current event count or a guarantee that every event will be retrieved. A heavily populated individual window can still reach the paging limit.

## PostgreSQL publishing

In [PR #89](https://github.com/HackYourFutureProjects/c55-final-project-group-A/pull/89), I changed the publish step from dropping and renaming tables to loading staging and refreshing the existing table inside a transaction.

The backend had a view that depended on `analytics.external_events`. Keeping the target table preserved the dependent view, permissions and indexes. The transaction protects the published data from an incomplete refresh; using `DROP CASCADE` would have removed dependencies instead of fixing the contract.

The current publisher has continued to evolve through team contributions. This PR identifies the refresh change I authored, rather than claiming every part of the current publishing implementation.

## Airflow integration

I connected backend publishing to the data workflow in [PR #76](https://github.com/HackYourFutureProjects/c55-final-project-group-A/pull/76). This brought the published event data into the application's PostgreSQL database through the pipeline's orchestration.

The team pipeline orders ingestion, landing-file discovery, dbt build and publishing. Airflow and the provided infrastructure form the foundation; my PR is evidence for my publishing integration work.

## Health dashboard

In [PR #117](https://github.com/HackYourFutureProjects/c55-final-project-group-A/pull/117), I customized the supplied optional Streamlit starter into a read-only health page for the event pipeline.

It covers published event counts, price coverage, freshness and landing-file information. The intent is to make data health visible to the team while keeping task logs and run history in Airflow.

Other PRs subsequently refined the dashboard and its metrics. The linked PR documents the customization I contributed.

## Team and foundation

The project includes frontend, backend and data work by several contributors, supported by HYF mentors. I worked alongside Pavel Tisner on data engineering.

Frontend, backend, baseline infrastructure and other data transformations retain their own authorship. The team's names and mentor credits remain in the [main README](../README.md#team).

The linked PRs are merged contributions to the canonical team repository. Their reported test results and development outputs are historical evidence from the relevant changes. They do not establish the current availability of the team's hosted services.
