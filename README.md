# plyzen Instrumentation Guide

## Overview

This document provides a comprehensive guide to collecting events from your software delivery pipeline and ingesting them into [plyzen](https://plyzen.io).

The data is the foundation for deriving the key [DORA metrics](https://dora.dev/guides/dora-metrics-four-keys/) to analyze and optimize your software delivery performance for speed, quality, and efficiency.

## What is Instrumentation?

Instrumentation involves placing webhooks at specific points within your delivery pipeline to capture events. These events are then sent to the plyzen web API, where they are processed. The results are immediately presented to you in the web dashboard.

<img width="644" alt="how it works" src="https://github.com/user-attachments/assets/02d5af05-81db-46ef-954a-c39e09f11ade">

## Why Webhooks?

Webhooks provide a lightweight and efficient mechanism for capturing and transmitting real-time data from your delivery pipeline. They enable seamless integration with existing tools and workflows, ensuring that relevant events are captured without significant overhead. This method is flexible and universally supported, making it tool agnostic. Most tools support webhooks, and they can be easily leveraged for custom tools, scripts, and processes. The goal is always to achieve minimally invasive instrumentation while maintaining maximum flexibility for individual needs and specific requirements.

## Hello World Example

To give you a better idea of what it looks like, let's submit a basic deployment event:

```shell
curl -X POST -H "Content-Type: application/json" -H "Authorization: YOUR_API_KEY" -d '{
  "artifact": "hello-world-app",
  "version": "1.0.0",
  "environment": "prod",
  "activity": "deployment",
  "event": "finish",
  "timestamp": "'"$(date -u +%Y-%m-%dT%H:%M:%SZ)"'",
  "result": "success"
}' https://in.plyzen.io
```

> **Note**: Ensure you replace `YOUR_API_KEY` with your actual plyzen API key.

Explanation:

* **artifact**: The deployment artifact named 'hello-world-app' is deployed.
* **version**: The version of the artifact being deployed.
* **environment**: We have deployed to the prod environment.
* **activity**: The activity type is a deployment.
* **event**: We are signaling that the deployment has finished.
* **timestamp**: The time at which the deployment was finished.
* **result**: The deployment was successessful.

If everything went well, the api returns with:

```json
{"result":"success"}
```

You will then find an entry for the artifact in the plyzen web app.

![image](https://github.com/user-attachments/assets/17fccd66-71ff-4200-9af5-4cd774fd9ec5)

The Deployment Frequency for `hello-world-app` is 1 and the Change Fail Percentage is 0%.

![image](https://github.com/user-attachments/assets/001b0d63-493e-4e36-b997-02f4368ff4bf)

## Start and Finish Events

Each activity in plyzen should have a corresponding start and finish event. The finish event can be sent without a preceding start event, indicating completion with `"result": "success"` or `"result": "failure"`. However, if a start event is sent without a corresponding finish event, plyzen will assume after a certain period of time that the activity has not been completed and has failed.

## Importance of Versioning

Versioning of artifacts is crucial for correlating events in plyzen. In software development, versioning is a best practice to uniquely identify releases, builds, or changes. For plyzen, the method of versioning doesn't matter as long as events can be distinctly associated with a version.

Though plyzen is tolerant (e.g., handling repeated events with the same version like Maven's SNAPSHOT concept), unclear versioning can affect metric precision. While plyzen doesn’t enforce a specific versioning scheme (e.g., Semver), it's essential that values are unique. plyzen determines the sequence of events by their timestamps, not by version order.

Adopting a "build once, run anywhere" approach is recommended. plyzen can handle repeated builds and re-deployments of the same version, adhering to its principle of minimal invasiveness, without requiring specific adjustments to your processes and pipelines. However, clear and unique versioning is always advisable as a general best practice.

## Best Practice: Go Step-by-Step. Start with the Quick wins.

The following graphic provides a visual roadmap for effective instrumentation.

<img width="1643" alt="image" src="https://github.com/user-attachments/assets/87e4a4a6-9ae5-4659-a827-119e9eaf0fd9">

Start with high-impact, low-effort steps – such as instrumenting production deployments – to quickly gain insight into your DORA metrics. As you progress, refine your setup by adding more events, such as build and monitoring events, to improve data accuracy and quality. Finally, expand your instrumentation to cover the entire value stream for comprehensive visibility and deeper insights into your software delivery process. This approach balances quick wins with long-term improvements.

## Measuring Deployment Frequency (DF)

Deployment Frequency (DF) is a key DORA metric that reflects how often your team deploys code to production. It’s important because it indicates the pace at which you’re delivering value to your end users.

Key Considerations

* **End-to-End Focus**: DORA metrics are end-to-end, so ***DF only counts deployments to production ("environment": "prod")***. While plyzen only considers production deployments for DF, you can still track deployments to other environments (e.g., dev, test, qa) using custom environment names.
* **Tracking Change Failures**: By setting "result": "success" or "fail", plyzen also tracks the Change Failure Rate, another DORA metric.
* **Impact on Lead Time**: DF data also helps measure Lead Time as the duration between two consecutive production deployments. This can later be refined for more precision.
* **Mean Time to Restore (MTTR)**: plyzen uses the data from successful and failed deployments to calculate MTTR, which measures the time it takes to restore service after a failed deployment. MTTR is determined by the time between a failed deployment ("result": "fail") and the next successful deployment ("result": "success").

## Measuring Lead Time (LT)

Lead Time (LT) is a critical DORA metric that measures the time it takes from a code change to production deployment. It indicates the speed at which you can deliver features or fixes to your users.

Key Considerations

* **Basic LT Measurement**: With only production deployment instrumentation (see [Measuring Deployment Frequency (DF)](#measuring-deployment-frequency-df) above), you get a baseline measurement of LT.
* **Enhanced Precision with Build Instrumentation**: By adding build instrumentation, plyzen can track when a version of an artifact was built, typically aligning closely with the time of the corresponding code change (triggered by a push to SCM under CI principles).
* **Comprehensive Lead Time Calculation**: plyzen does not just measure the time from building a version to deploying that same version (which would only capture the pipeline's throughput time—an interesting metric in itself). Instead, plyzen also includes all builds leading up to the new release version. The calculation starts from the moment the last successfully deployed version was built and includes all subsequent builds. This approach ensures:
  1. Inclusion of All Code Changes: LT considers all code changes leading to a successful release.
  2. Accurate Start Time: If an artifact is not continuously developed, the LT starts from the moment development resumes on that version.

### Basic Build Event Example

To track a build, you send events with activity type `build`. Here's a basic example:

**Start Event**
```json
{
  "artifact": "hello-world-app",
  "version": "1.0.0",
  "environment": "ci",
  "activity": "build",
  "event": "start",
  "timestamp": "2024-08-10T14:23:00Z"
}
```

**Finish Event**
```json
{
  "artifact": "hello-world-app",
  "version": "1.0.0",
  "environment": "ci",
  "activity": "build",
  "event": "finish",
  "timestamp": "2024-08-10T14:25:00Z",
  "result": "success"
}
```

This setup allows plyzen to accurately calculate Lead Time by correlating the build information with deployment events.

## Change Failure Percentage (CFP)

Change Failure Percentage (CFP) measures the rate at which changes fail during or after deployment. plyzen uses three signals to calculate CFP:

1. **Deployment Failure**: A deployment event with `"result": "failure"`.
2. **Post-Deployment Test Failure**: Any activity of type `"test"` that fails after the same version of an artifact is deployed in the same environment.
3. **Alarm Activity**: An activity of type `"alarm"` associated with the same version of the deployed artifact.

Example CFP Scenarios

| Artifact         | Version | Deployment Result | Smoke Test Result | Alarm occurred? | Ongoing CFP Calculation |
|------------------|---------|-------------------|-------------------|-----------------|-------------------------|
| hello-world-app  | v1      | success           | success           | no              | 0%                      |
| hello-world-app  | v2      | success           | failure           | no              | 50%                     |
| hello-world-app  | v3      | success           | success           | yes             | 66.7%                   |

These examples show how plyzen aggregates these signals to calculate CFP. Note that due to the end-to-end nature of the DORA metrics, only events in the "prod" environment count toward the metric.

## Mean Time to Restore (MTTR)

*tbd*

## Aggregated Product Metrics

In plyzen, you can also measure DORA metrics at the product level, beyond individual artifacts. This approach aggregates events from multiple artifacts to provide a comprehensive view of the entire product's performance.

Key Considerations

* **Product-Level View**: By defining a product as a collection of artifacts, plyzen allows you to assess DORA metrics at a higher level of abstraction.
* **Event Aggregation**: plyzen aggregates events across all artifacts associated with the product, giving you insights into the overall speed and quality of the product delivery, rather than just its individual components.

By utilizing aggregated product metrics, you can better understand the impact of your deployments and development activities on the final product delivered to your users.

*to be continued ...*

## Idempotence for Replayed Events

Idempotence is crucial when replaying events to ensure that repeated events do not alter the state incorrectly. In plyzen, if an event is replayed—whether due to retries or intentional resubmission—the system processes it in a way that prevents duplicate effects. This means that if the semantically same event is received multiple times, plyzen will recognize it and skips repeated processing to avoid unintended effects and maintain data consistency.

## Upsert/Merge and Amend Events

plyzen supports the ability to upsert, merge, or amend events, which is vital for maintaining accurate tracking data. Using the `correlationId` property with a unique and reproducable value, multiple records can be associated with the same activity, allowing you to update or correct previously sent data. This is particularly useful when dealing with complex workflows where data might need to be sent in chunks or when revising earlier event data. By leveraging this feature, you can ensure that your event history remains precise and reflective of actual activities.

### Use Case Examples

#### Upsert an Aggregated Deployment Activity

1. Start an "empty" aggregated product deployment.
2. Later add the artifacts and versions deployed.

> **Note** It is not necessary to repeat already sent data. All you need to repeatingly provide is the `correlationId`.

#### Amend Previously Ingested Data

E.g. if you imported data manually and made a mistake with the timestamp, you can repeat the submission with correct timestamps.

## Event Message Schema Reference: Basic and Advanced

*tbd*
