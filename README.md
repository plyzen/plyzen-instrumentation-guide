# plyzen Instrumentation Guide

## Overview

This document provides a comprehensive guide to collecting events from your software delivery pipeline and ingesting them into [plyzen](https://plyzen.io).

The data is the foundation for deriving the key [DORA metrics](https://dora.dev/guides/dora-metrics-four-keys/) to analyze and optimize your software delivery performance for speed, quality, and efficiency.

### tl;dr

To instrument your software delivery pipeline with plyzen, `POST` event messages in JSON format to the API endpoint. Use activity types such as `build`, `deployment`, `test`, and `alarm` to capture different stages of the process that contribute to the metrics. Two message schemas are available – [basic](#basic-ingest-schema) and [advanced](#advanced-ingest-schema) – to accommodate different use cases.
There are [#instrumentation-utilities](utilities and examples) to speed things up if you use Gitlab, Jenkins, or custom scripts.

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
* **Mean Time to Recovery (MTTR)**: plyzen uses the data from successful and failed deployments to calculate MTTR, which measures the time it takes to restore service after a failed deployment. MTTR is determined by the time between a failed deployment ("result": "fail") and the next successful deployment ("result": "success").

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

## Change Fail Percentage (CFP)

Change Fail Percentage (CFP) measures the rate at which changes fail during or after deployment. plyzen gives you the ability to send three types of signals to calculate CFP:

1. **Deployment Failure**: A deployment event with `"result": "failure"`.
2. **Post-Deployment Test Failure**: Any activity of type `"test"` that fails after the same version of an artifact is deployed in the same environment.
3. **Alarm Activity**: An activity of type `"alarm"` associated with the same version of the deployed artifact.

Any combination of these signals can be implemented. If at least one event signals a failure, it counts toward the CFP.

Example CFP Scenarios

| Artifact         | Version | Deployment Result | Smoke Test Result | Alarm occurred? | Ongoing CFP Calculation |
|------------------|---------|-------------------|-------------------|-----------------|-------------------------|
| hello-world-app  | v1      | success           | success           | no              | 0%                      |
| hello-world-app  | v2      | failure           | not started       | no              | 50%                     |
| hello-world-app  | v3      | success           | failure           | no              | 66.7%                   |
| hello-world-app  | v4      | success           | success           | yes             | 75%                     |

These examples show how plyzen aggregates these signals to calculate CFP. Note that due to the end-to-end nature of the DORA metrics, only events in the "prod" environment count toward the metric.

## Mean Time to Recovery (MTTR)

Mean Time to Recovery (MTTR) measures the time it takes to recover from a (change) failure and restore a stable service in production. plyzen calculates MTTR based on the signals used for CFP:

* **Successful (Re)Deployment**: The time to recovery begins when a deployment fails or when a post-deployment test or alarm indicates an issue. It ends when a new (or the same) version is successfully deployed to production, the tests pass (if any), and the alarm is resolved (if any).
* **Test Failure Recovery**: If the initial deployment was successful but the test failed, the time to recovery ends with a successful retest.
* **Alarm Recovery**: If an alarm was signaled, time to recovery ends when the alarm is resolved ("activity": "alarm", "event": "finish").

This approach ensures that MTTR reflects the true recovery time required to restore a stable service state in production.

## Aggregated Product Metrics

In plyzen, you can also measure DORA metrics at the product level, beyond individual artifacts. This approach aggregates events from multiple artifacts to provide a comprehensive view of the entire product's performance.

Key Considerations

* **Product-Level View**: By defining a product as a collection of artifacts, plyzen allows you to assess DORA metrics at a higher level of abstraction.
* **Event Aggregation**: plyzen aggregates events across all artifacts associated with the product, giving you insights into the overall speed and quality of the product delivery, rather than just its individual components.
  * **Lead Time**: The Lead Time of a product is calculated as the average of the Lead Times of the artifact versions that make up the product releases.
  * **Deployment Frequency**: The Deployment Frequency is determined by the frequency of production deployment events associated with the product name, not by averaging the deployment frequencies of the individual artifacts.
  * **Change Fail Percentage**: You can explicitly specify a result for the entire product deployment. If no result is specified, plyzen automatically determines the overall result based on the results of the included artifacts.
    * Success: If all artifacts are deployed successfully.
    * Failure: If the deployment result of all artifacts is failure.
    * Diverse: If the result for some artifacts is success while for others is others failure.

    It works similarly for Tests and Alarms (see CFP for artifacts for the concept), except that you specify the product name and a list of affected artifacts (analogous to the deployment example shown below).
  * **Mean Time to Restore**: MTTR for products works similarly to artifacts. The difference is that the events include a product name and a list of affected artifacts. The MTTR is calculated based on the time from a failure until the product or all of its artifacts are restored to a stable state.

By utilizing aggregated product metrics, you can better understand the impact of your deployments and development activities on the final product delivered to your users.

### Product Metrics Example

To illustrate how plyzen aggregates metrics at the product level, consider a product called "SuperApp", which consists of three artifacts: "auth-service", "user-interface", and "data-processor." Each artifact has its own versioning and deployment cycle. They can be deployed independently or combined as a deployment monolith or something in between.

Here’s how you might structure a JSON event for the combined "SuperApp" deployment in the advanced format:

```json
{
  "activities": [
    {
      "correlationId": "superapp-prod-deploy-1001",
      "name": "prod-deploy",
      "type": "deployment",
      "events": [
        {
          "type": "finish",
          "timestamp": "2024-08-10T15:00:00Z"
        }
      ],
      "result": "success",
      "product": {
        "name": "SuperApp"
      },
      "artifacts": [
        {
          "name": "auth-service",
          "version": "1.2.0",
          "result": "success"
        },
        {
          "name": "user-interface",
          "version": "3.5.1",
          "result": "success"
        },
        {
          "name": "data-processor",
          "version": "2.0.0",
          "result": "success"
        }
      ],
      "environment": {
        "name": "prod"
      }
    }
  ]
}
```

## Idempotence for Replayed Events

Idempotence is crucial when replaying events to ensure that repeated events do not alter the state incorrectly. In plyzen, if an event is replayed—whether due to retries or intentional resubmission—the system processes it in a way that prevents duplicate effects. This means that if the semantically same event is received multiple times, plyzen will recognize it and skips repeated processing to avoid unintended effects and maintain data consistency.

## Upsert/Merge and Amend Events

plyzen supports the ability to upsert, merge, or amend events, which is vital for maintaining accurate tracking data. Using the `correlationId` property with a unique and reproducable value, multiple records can be associated with the same activity, allowing you to update or correct previously sent data. This is particularly useful when dealing with complex workflows where data might need to be sent in chunks or when revising earlier event data. By leveraging this feature, you can ensure that your event history remains precise and reflective of actual activities.

Always use the POST HTTP method.

### Use Case Examples

#### Upsert an Aggregated Deployment Activity

1. Start an "empty" aggregated product deployment.
2. Later add the artifacts and versions deployed.

> **Note** It is not necessary to repeat already sent data. All you need to repeatingly provide is the `correlationId`.

#### Amend Previously Ingested Data

E.g. if you imported data manually and made a mistake with the timestamp, you can repeat the submission with correct timestamps.

## Event Message Schema Reference: Basic and Advanced

plyzen supports two schemas for ingesting events:

1. the **Basic Ingest Schema** and
2. the **Advanced Ingest Schema**.

These schemas define the structure of the JSON payloads that are sent to the plyzen API to capture various activities within your delivery pipeline.

### Basic Ingest Schema

The Basic Ingest Schema is designed for simple use cases and is perfect for most scenarios. It contains the essential fields in a flat JSON object to keep it simple and stupid (KISS). We recommend using it unless you have more sophisticated requirements.

**Example:**
```json
{
  "artifact": "example-artifact",
  "version": "1.0.0",
  "activityType": "deployment",
  "environment": "prod",
  "event": "finish"
}
```
This example shows a basic schema message with the minimum required fields. It signals the end of an artifact deployment to production.

For a fully documented reference of the basic message format, see [`basic-ingest-schema.json`](basic-ingest-schema.json).

### Advanced Ingest Schema

The Advanced Ingest Schema provides greater flexibility and detail to support complex scenarios, such as batch submission of activities. It allows events to be aggregated across different artifacts and is better suited for upserting, merging, and modifying event data.

**Example 1: Minimal Advanced Message**

```json
{
  "activities": [
    {
      "correlationId": "deploy-1001",
      "type": "deployment",
      "events": [
        {
          "type": "finish"
        }
      ],
      "artifacts": [
        {
          "name": "example-artifact",
          "version": "1.0.0",
          "environment": {
            "name": "prod"
          }
        }
      ]
    }
  ]
}
```
This message is semantically identical to the basic schema example provided earlier, using the advanced format.

**Example 2: Extended Advanced Message**

```json
{
  "activities": [
    {
      "correlationId": "deploy-1001",
      "type": "deployment",
      "events": [
        {
          "type": "finish"
        }
      ],
      "artifacts": [
        {
          "name": "example-artifact",
          "version": "1.0.0",
          "environment": {
            "name": "prod"
          }
        }
      ]
    },
    {
      "correlationId": "superapp-prod-deploy-2001",
      "type": "deployment",
      "events": [
        {
          "type": "start",
          "timestamp": "2024-08-10T16:55:00Z"
        },
        {
          "type": "finish",
          "timestamp": "2024-08-10T17:00:00Z"
        }
      ],
      "result": "success",
      "product": {
        "name": "SuperApp"
      },
      "artifacts": [
        {
          "namespace": "core-services",
          "name": "auth-service",
          "version": "1.2.0",
          "result": "success"
        },
        {
          "namespace": "frontend",
          "name": "user-interface",
          "version": "3.5.1",
          "result": "success"
        },
        {
          "namespace": "backend",
          "name": "data-processor",
          "version": "2.0.0",
          "result": "success"
        }
      ],
      "environment": {
        "name": "prod"
      }
    }
  ]
}

```
This example demonstrates a batch of two activities: one identical to the minimal message above and another representing a product aggregation with three artifacts.

For a fully documented reference of the basic message format, see [`advanced-ingest-schema.json`](advanced-ingest-schema.json).

## Instrumentation Utilities

To further streamline your plyzen instrumentation, several utilities are available in the [plyzen GitHub repositories](https://github.com/plyzen). These tools provide scripts, helpers, and further information to automate the creation and sending of events in various formats, including Bash and Groovy implementations for use in Gitlab and Jenkins. By using these utilities, you can quickly set up and manage your instrumentation to ensure consistent and accurate data collection throughout your software delivery pipeline.
