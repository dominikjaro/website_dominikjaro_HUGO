---
title: "🚀 Migration Project: Modernizing Content Delivery Feeds -- NOT FINISHED YET TBC"
date: 2026-06-30
description: "The legacy infrastructure relies on a monolith platform to generate and distribute critical data feeds (e.g., Google Merchant Reviews/Products)."
categories: [""]
tags: ["GCP"]
image: ""
---

## 🏗️ Context & Architecture

Historically, these files were generated locally on a Monolith VM via cron jobs and synced over to a separate, dedicated **FTP VM**. External clients (such as Google) then authenticated via a highly trusted **SFTP account** to collect these files directly from the VM's local file system.

### 📊 Scale of the Operation

- **Feed A (Google Merchant Reviews):** ~2,000 files per generation. (Subject of Phase 1 Migration)

- **Feed B (Other Content Feeds):** ~8,000+ folder and many files per generation, deeply nested in complex subdirectories.

## ☸️ Phase 1: The Kubernetes Plan

To break apart the monolith, we initiated a plan to migrate Feed A (~2,000 files) over to a containerized **Kubernetes (K8s) CronJob**.

### 🛠️ The Proposed K8s & Storage Architecture
1. **Init Container:** Generates the ~2,000 feed files and dumps them into a shared `emptyDir` volume.
2. **Main Container:** Picks up the files from the `emptyDir` and uploads them to a newly created Google Cloud Storage (GCS) bucket using `gcloud storage cp`.
3. **The FTP VM Bridge:** On the FTP VM, we mounted the GCS bucket locally using `gcsfuse`. The plan was to alter the Google SFTP user’s home directory and permissions to point straight into this mounted fuse directory, allowing Google to pull files straight from the cloud bucket.

## 🚨 The Incident & Rollback

### 🧪 The Pre-Rollout Test

Before letting the K8s CronJob loose, we executed a phased testing approach: we manually copied the files from the old local directory to the new `gcsfuse` directory to verify if the Google SFTP user could collect them.

### ⚠️ The Issue

Initial validation showed that the Google SFTP client could authenticate successfully. However, file collection failed completely. Because `gcsfuse` has to translate POSIX file system calls (like ls or recursive directory lookups) into HTTP cloud API calls sequentially, scanning a folder containing thousands of files proved to be an incredibly high-latency operation. The connection hit a timeout before Google's SFTP client could even finish listing and fetching the feeds.

### ↩️ The Action

**IMMEDIATE ROLLBACK.** The SFTP user's path was reverted back to the native local file system. Normal file collection resumed immediately, preventing any production data delivery blockage.

## 📋 The Revised Strategy (Tactical Solution)

To avoid blocking the containerization rollout of the file generation service, we decoupled the K8s migration from the SFTP file system migration.

## ⚙️ Current Working Architecture

Instead of forcing the SFTP client to read over a network file system (`gcsfuse`), we kept the collection endpoint on the FTP server's trusted, high-performance local disk, and introduced a unidirectional sync script.

- K8s CronJob Side: Successfully containerized. It runs, generates the files, and cleanly pushes them to a dedicated GCS bucket.
- FTP VM Side: We deployed a local Bash script triggered by a cron job running under a dedicated service user (syncuser). The script leverages `gcloud storage rsync` in production environments to pull files down from the GCS bucket back onto the local file system.

`[ K8s CronJob ] ---> ( GCS Bucket ) ---> [ FTP VM Cron (gcloud storage rsync) ] ---> [ Local FS ] ---> [ Google SFTP Client ]`

This drastically simplified the deployment blast radius while still allowing us to deprecate the file-generation code on the legacy monolith VM.

## 🧠 Key Takeaways & Struggles

- **Object Storage vs. POSIX Filesystems:** `gcsfuse` is a fantastic tool for low-throughput or highly specific workloads, but it is notoriously performant-deficient when a client expects rapid, directory-wide file listing and streaming of thousands of files simultaneously.

- **The Importance of Phased Testing:** Testing the permission boundaries and file collection before switching the generation mechanism to K8s saved us from a blind production outage.

## 🗺️ Moving Ahead (Future Roadmap)

The long-term goal remains to completely decommission both the monolith VM and the dedicated FTP server. To achieve this, the next phase will require a client-facing architectural shift:

1. **Direct Cloud Collection:** Investigate how clients (like Google) can authenticate and collect files directly from dedicated, restricted GCP Buckets rather than routing through an intermediate SFTP server.

2. **The Shared-Directory Challenge:** Because different clients collect different combinations of nested folders and files (some targeting the 2,000 file feed, others targeting the 8,000+ nested feed), we cannot migrate clients one-by-one seamlessly if they rely on a single directory footprint. We must coordinate a strategy to cut over all dependent client feeds simultaneously when shifting away from the sftp system.