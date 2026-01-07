---
title: "DICOM Server rs v1.0.0 published"
date: 2026-01-06T15:21:56+08:00
weight: 1
keywords: "DICOM-WEB, medical imaging, healthcare cloud, DICOM storage, cloud DICOM platform, DICOM services"
description: "DiCOM Server rs v1.0.0 published"
draft: false
tags: ["DICOM-WEB", "medical imaging", "healthcare cloud", "DICOM storage", "cloud DICOM platform"]
categories: ["Medical Imaging", "DICOM Development", "Healthcare Technology"] 
---

## Announcing dicom-server-rs v0.1.0: A High-Performance DICOM Store SCP in Rust

We are excited to announce the initial release of dicom-server-rs, a lightweight, high-performance DICOM server (Store SCP) built entirely in Rust.

In the world of medical imaging, reliability and speed are non-negotiable. By leveraging the Rust ecosystem, dicom-server-rs provides a modern alternative for receiving and managing DICOM files with a focus on safety and efficiency.

### 🚀 Key Features

- Native Rust Implementation: Built using the dicom-rs ecosystem for robust parsing and protocol handling.

- Store SCP Support: Fully supports the C-STORE operation, allowing it to act as a provider for imaging modalities (CT, MRI, X-ray) and PACS systems.

- Asynchronous Architecture: Utilizing tokio to handle multiple concurrent associations without blocking.

- Minimal Footprint: Designed to be lightweight and easy to deploy via binary or container.

### 🛠 Why Rust for DICOM?

DICOM files can be massive, and network associations often involve complex state machines. Traditional implementations often struggle with memory overhead or vulnerability to buffer overflows. dicom-server-rs solves this by:

- Memory Safety: Eliminating common memory-related bugs at compile time.

- Zero-Cost Abstractions: Providing high-level API ergonomics without sacrificing the performance of C/C++.

- Concurrency: Effortlessly managing simultaneous incoming streams from multiple AE (Application Entity) titles.


### 📦 Quick Start
To get started with v0.1.0, you can clone the repository and run the server:

 

- Clone and Build
```bash
git clone https://github.com/momostarsky/dicom-server-rs.git
cd dicom-server-rs
cargo build --release
```

- Prepare Database and Storage Service 
```bash
cd dicom-server-rs
mkdir ~/verify-dicom
cp -r Docs/Install ~/verify-dicom/
cd ~/verify-dicom/Install
docker-compose up -d
```

- Run Server

  - wado-server 
  - wado-storescp 
  - wado-consumer 
  - wado-webworker	 


## how to cofigure application.XXX.json

Refrences: [Install](https://github.com/momostarsky/dicom-server-rs/blob/main/Docs/Readme.md) 



## 🗺 Roadmap for v0.1.x and Beyond
While v0.1.0 focuses on the core Store SCP functionality, we have ambitious plans for the future:

-   Query/Retrieve (C-FIND/C-MOVE): Implementing full PACS capabilities.

-   Web Dashboard: A simple UI to monitor incoming traffic and storage status.
 
-   Custom Storage Backends: Support for S3 or other cloud storage providers.

-   OAuth2: Accessing DICOM files through OAuth2 for secure authorization.

-   Comprensive Documentation: A comprehensive guide for users and developers.

## 🤝 Join the Community
This project is open-source and we welcome contributions! Whether you are a medical imaging professional, a Rust enthusiast, or a student looking to learn about healthcare tech, your input is valuable.

- Star the repo: [DicomServerRs](https://github.com/momostarsky/dicom-server-rs)

- Open an issue: Found a bug or have a feature request? Let us know.

- Submit a PR: We love code contributions!

Would you like me to refine this into a shorter Twitter/X announcement or a more technical deep-dive for a site like Dev.to?