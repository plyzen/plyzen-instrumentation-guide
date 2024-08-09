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




