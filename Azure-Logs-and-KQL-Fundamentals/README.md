# Azure Logging and KQL Fundamentals

## Project Overview

This project introduces the basics of working with Azure logs and writing simple Kusto Query Language (KQL) queries. The goal is to understand how Azure collects log data, how that data is structured, and how KQL is used to search and filter through it.

The environment used in this project is a live, community-based Azure lab where users deploy and interact with virtual machines. These actions generate real log data inside Azure, including events such as virtual machine creation, sign-ins, network activity, and system changes. This provides a realistic dataset to practice querying and analyzing cloud activity.

Throughout this project, I will explain the fundamentals of KQL and demonstrate simple commands to explore log data. These examples will serve as the foundation for more advanced queries and investigations in later projects.

---

## What is KQL?

**Kusto Query Language (KQL)** is a query language used to search, analyze, and explore large amounts of data stored in Azure services such as Azure Monitor, Log Analytics, and Microsoft Sentinel. It is designed specifically for working with log and telemetry data, making it a core tool for cloud monitoring and security investigations.

KQL works by running queries against tables of data, where each table contains structured logs (for example, sign-in events, virtual machine activity, or network traffic). Instead of manually searching through raw logs, KQL allows you to filter, sort, and summarize data using readable commands like `where`, `project`, `summarize`, and `sort by`. This makes it much easier to quickly find relevant events in large datasets.

KQL is important because modern cloud environments generate massive amounts of log data every second, which would be impossible to analyze manually. Security analysts, cloud engineers, and SOC teams rely on KQL to investigate incidents, detect suspicious activity, and monitor system health. It is especially valuable in tools like Microsoft Sentinel, where fast and accurate log analysis is essential for identifying threats and responding to security events.

---

## Simple KQL Commands

To start exploring logs, we begin with the `SecurityEvent` table.


