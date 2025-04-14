# Docs

A collection of technical guides and documentation.

## Table of Contents
- [Docs](#docs)
  - [Table of Contents](#table-of-contents)
  - [Install Lego ACME client for Windows {#lego-windows}](#install-lego-acme-client-for-windows-lego-windows)
  - [Creating Additional ColdFusion Instances in a Secure Baseline Environment {#coldfusion-instances}](#creating-additional-coldfusion-instances-in-a-secure-baseline-environment-coldfusion-instances)
  - [Git stash is a powerful feature... {#git-stash}](#git-stash-is-a-powerful-feature-git-stash)

## Install Lego ACME client for Windows {#lego-windows}

This guide provides instructions for downloading and installing the Lego ACME client alongside CommandBox on Windows systems. The document includes a PowerShell script that automatically fetches the latest version of Lego, extracts it to the appropriate location, and configures your system PATH. Also included are troubleshooting tips for common DNS providers like Cloudflare and EasyDNS.

[Install Lego ACME Client for Windows](https://github.com/murpg/Docs/blob/main/installLegoWindows.md)

## Creating Additional ColdFusion Instances in a Secure Baseline Environment {#coldfusion-instances}

This comprehensive guide details the process of creating additional ColdFusion instances in a security-hardened (secure baseline) environment, with a particular focus on environments using custom service accounts instead of the default System account. The document provides detailed PowerShell scripts for managing services, setting up registry permissions, and handling the file structure required for new ColdFusion instances.

The guide is particularly valuable for system administrators and DevOps engineers working in secure environments, as it includes extensive error handling, verification steps, and best practices. It covers everything from basic setup to troubleshooting common issues, with special attention paid to service account permissions and proper service management in a security-baseline environment.

[Creating Additional ColdFusion Instances](https://github.com/murpg/Docs/blob/main/add-another-instance-coldfusion.md)

## Git stash is a powerful feature... {#git-stash}
Git stash is a powerful feature that temporarily shelves (or stashes) changes you've made to your working copy so you can switch to another task, and then come back and re-apply them later.

[Git Stash: A Comprehensive Guide](https://github.com/murpg/Docs/blob/main/git-stash.md)