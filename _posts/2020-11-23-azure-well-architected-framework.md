---
title: Azure Well-Architected Framework
date: 2020-11-23 00:00:00 +0000
description: Following industry standards and terms, the Azure Well-Architected Framework provides a set of Azure architecture best practices that support your cloud solution success.
categories: [Miscellaneous]
tags: [Well-Architected Framework]
header:
 teaser: "/assets/img/posts/teasers/advisor.png"
permalink: /misc/azure-well-architected-framework/
excerpt: Following industry standards and terms, the Azure Well-Architected Framework provides a set of Azure architecture best practices that support your cloud solution success. The five pillars help you effectively and consistently optimize your workloads against Azure best practices and the specific business priorities relevant to you or your customers' cloud journey.
---
### Well-Architected at Microsoft
Well-Architected at Microsoft means achieving business objectives with an ecosystem of best practices, assessments, and technical guidance that includes:
* [Azure Well-Architected](https://docs.microsoft.com/en-us/azure/architecture/framework/) Framework enables you to achieve your business objectives, operationalizing how to design, build, and optimize cloud solutions.
* [Azure Well-Architected Review](https://docs.microsoft.com/en-us/assessments/?id=azure-architecture-review&mode=pre-assessment) evaluates workloads with gap analysis and identifies areas of where to focus your optimization efforts.
* [Reference Architectures](https://docs.microsoft.com/en-us/azure/architecture/browse) help you to build and deploy any workload to scale and meet your business needs.
* [Design principles](https://docs.microsoft.com/en-us/azure/architecture/guide/design-principles/) based on proven practices and design patterns from successful customer and partner deployments.
* [Documentation](https://docs.microsoft.com/en-us/azure/?product=featured) provides answers to your how-to questions.
* [Azure Advisor](https://docs.microsoft.com/en-us/azure/advisor/) gives you prioritized recommendations on your running workloads.

![My View]({{ "/assets/img/posts/well-architected-framework/wellArchitectedAtMicrosoft.png" | relative_url }})
### Azure Well-Architected Framework
Following industry standards and terms, the Azure Well-Architected Framework provides a set of Azure architecture best practices that support your cloud solution success. The framework is divided into five pillars of architectural best practices: cost management, operational excellence, performance efficiency, reliability, and security. These pillars help you effectively and consistently optimize your workloads against Azure best practices and the specific business priorities relevant to you or your customers' cloud journey.

* Cost-Optimization: Design "pay-as-you-go" cost-effective workloads aligned with business objectives/ROI while maintaining a budget
* Operational Excellence: Design reliable, predictable, automated deployments, with monitoring, performance management, and extensive automated and manual testing from an infrastructure and application perspective
* Performance Efficiency: Lower maintenance costs, improve user experience, and increase agility by architecting solutions with scalability baked-in. Move to PaaS by default to use built-in scaling functionality.
* Reliability: Scale-out instead of scaling up with expensive hardware, and build reliability across deployments with resilient, HA applications, and failure mode analysis
* Security: Build with security by design--to provide confidentiality, integrity, and availability assurances against deliberate attacks and abuse of your valuable data and system

Each of the pillars is discussed in detail in the framework. 

### Azure Well-Architected Review
The Azure Well-Architected Review is designed to help you evaluate your workloads against the latest set of Azure best practices. It provides you with a suite of actionable guidance that you can use to improve your workloads in the areas that matter most to your business. You can evaluate each workload against only the pillars that matter for that workload, so when evaluating one of your mission-critical workloads, you might examine the reliability, performance efficiency, and security first and then later come back and look at the other pillars to improve your operational efficiency and cost footprint. Key points are:

* It has a series of questions across the 5 pillars of Azure Well-Architected Framework, and responses to the questions evaluate key items needing attention for successful workload deployment and management.
* The results of the assessment provide actionable steps to consider implementing to improve the quality of the workload.
* This will help you overcome your hurdles effectively and build your workload with confidence.
* It also allows you to chose only one pillar or more, depending on where you want to focus.
* If you choose more than one, you get gauged results for each pillar, helping you to focus on the areas your workload needs more attention.
* The assessment will typically require about 20-25 mins to complete if you choose all 5 pillars.
* If you log in with your Microsoft account, you can save the assessment reports.
* The assessment report is exportable in Excel format.

### Conclusion
The Azure Well-Architected Framework is a great tool that can help you architecting a cloud solution. The framework is applicable for any new or existing workloads, either you plan to migrate or modernize your workload. 

If you have an existing workload, start with the Azure Well-Architected Review first. That will help you to evaluate your workloads against Azure best practices and provide you an overall score on the pillar you are evaluating against. Each of the recommendations is followed by a link to learn more about implementing the practice or mitigating the potential problem. Create tasks for those recommendations and prioritize accordingly. 

For new workloads, use the framework documentation to familiarize yourself with best practices before building your solution. The free learning path at Microsoft Learn called ["Build great solutions with the Microsoft Azure Well-Architected Framework"](https://docs.microsoft.com/en-us/learn/paths/azure-well-architected-framework/) can help you achieve that.  After finishing the high-level architecture, use the Azure Well-Architected Review tool to verify that you considered the framework's best practices. 

The Azure Well-Architected Review assessment is a continuous process. It is recommended to execute it every six months because the practices may change, and new products or features may be introduced.