---
title: "EntraOps Privileged EAM"
layout: splash
permalink: /entraops/
header:
  overlay_color: "#001B38"
  overlay_filter: "0.5"
  overlay_image: /assets/images/entraops/entraops_banner.png
  actions:
    - label: "Get it on GitHub"
      url: "https://github.com/Cloud-Architekt/EntraOps"
excerpt: "Community project to classify, identify and protect your privileges based on Enterprise Access Model (EAM)"
feature_row1:
  - image_path: /assets/images/entraops/setup_1-ghconfig.gif
    title: "Automated setup and designed for DevOps platforms"
    excerpt: "The core of EntraOps is a PowerShell module and works on any platform which supports PowerShell Core. It can be also executed locally within an interactive session. It's optimized for being used in a DevOps environment and includes an automated setup for GitHub and Federated Credentials. This also allows you to take advantage of Git, to compare changes of your privileged assets."
    url: "https://github.com/Cloud-Architekt/EntraOps?tab=readme-ov-file#using-entraops-with-github"
    btn_label: "Using EntraOps with GitHub"
    btn_class: "btn--primary"    
feature_row2:    
  - image_path: /assets/images/entraops/2-identify.png
    title: "Identify and classify your privileged assets"
    excerpt: "Get insights about privileged users, groups and workload identities with eligible, permanent, time-bounded and nested role assignments in Microsoft Entra. EntraOps uses a full customizable and flexible classification model. It provides templates (defined in JSON) to identify privileges and tiered administration based on [Enterprise Access Model](https://aka.ms/SPA)."
    url: "https://github.com/Cloud-Architekt/EntraOps/tree/main/Classification/Templates"
    btn_label: "Classification templates"
    btn_class: "btn--primary"
feature_row3:
  - image_path: /assets/images/entraops/setup_2-cpupdate.gif
    title: "Automatic update on Control Plane resources and scope"
    excerpt: 'An optional feature allows you to collect data from other sources to identify high-privileged assets. For example, using classification of critical assets in Microsoft Security Exposure Management. The scope of the privileged principals with access to those assets will be identified as Control Plane and therefore any delegation in Microsoft Entra to manage them. This allows to identify critical scoped role assignments on Groups, Service Principals or Administrative Units.'
    url: "https://github.com/Cloud-Architekt/EntraOps?tab=readme-ov-file#automatic-updated-control-plane-scope-by-entraops-and-other-data-sources"
    btn_label: "Details of Control Plane scope update"
    btn_class: "btn--primary"
feature_row4:
  - image_path: /assets/images/entraops/4-visualize.png
    title: "Analysis and visualization of privileged classification"
    excerpt: 'Ingestion to classified privileges with detailed data to Log Analytics Workspace or Sentinel WatchList is already integrated. This allows to use the EntraOps data for advanced hunting or other integration for entity enrichment. The provided KQL examples shows how to correlate classified privileges by EntraOps with Microsoft Security Exposure Management data. A workbook template to visualize results of all classified roles is available which allows to identify "tier breach". This allows you also to compare classification of privileged objects (based on custom security attribute) with their classified privileged access (identified by EntraOps).'
    url: "https://github.com/Cloud-Architekt/EntraOps?tab=readme-ov-file#entraops-integration-in-microsoft-sentinel"
    btn_label: "Microsoft Sentinel/XDR Integration"
    btn_class: "btn--primary"
---

{% include feature_row id="feature_row1" type="left" %}

{% include feature_row id="feature_row2" type="right" %}

{% include feature_row id="feature_row3" type="left" %}

{% include feature_row id="feature_row4" type="right" %}