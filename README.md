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
* **result**: The deplyment was successessful.

If everything went well, the api returns with:

```json
{"result":"success"}
```

You will then find an entry for the artifact in the plyzen web app.

![image](https://github.com/user-attachments/assets/17fccd66-71ff-4200-9af5-4cd774fd9ec5)

The Deployment Frequency for `hello-world-app` is 1 and the Change Fail Percentage is 0%.

![image](https://github.com/user-attachments/assets/001b0d63-493e-4e36-b997-02f4368ff4bf)

_To be continued..._
