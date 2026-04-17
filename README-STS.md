# AWS STS OIDC Configuration within secure AWS private government regions 

AWS recommends using temporary, limited-privilege credentials from its Security Token Service (STS) instead of long-term access keys, as this is an inherently more secure approach. While OpenShift Container Platform (since version 4.10) supports STS, its configuration in secure AWS regions, like GovCloud, presents a challenge. [see AWS official documentation for more information](https://docs.aws.amazon.com/STS/latest/APIReference/welcome.html)

The core issue is that STS requires OIDC provider documents to be publicly accessible via HTTPS. This is straightforward in commercial regions using public S3 buckets, but not possible in secure enclaves where resources are kept private. The standard CloudFront solution is not always available in these high-security environments.
This document explores various design options to solve this problem and provides a detailed, step-by-step implementation guide for a solution that uses an API Gateway to securely expose the necessary OIDC documents.

**The Security Advantage of AWS STS**
AWS Security Token Service (STS) allows applications to request temporary, limited-privilege credentials. This method is recommended by AWS over using permanent, long-term credentials (access and secret keys). The key security benefit is that these temporary credentials expire after a short period and are mapped to a role with only the minimum permissions required for a specific task, significantly reducing the risk associated with compromised keys.

**The Configuration Challenge in Secure Environments**
OpenShift Container Platform (v4.10+) can leverage the security of AWS STS. However, a critical dependency is that the cluster's OIDC provider documents must be publicly discoverable via an HTTPS endpoint.
  - In standard AWS regions: This is typically handled by hosting the OIDC documents in a public S3 bucket.
  - In secure AWS regions (e.g., GovCloud): Public buckets are not an option. The documents must reside in a private S3 bucket. The default solution provided by OpenShift's ccoctl tool uses CloudFront to expose the content, but CloudFront is not available in all secure government enclaves.

**Proposed Solution and Document Outline**
This creates a need for an alternative method to securely expose OIDC documents from a private S3 bucket over HTTPS. This document proceeds as follows:
1.	STS Design: We will explore and evaluate various technical solutions for hosting and publicly exposing the OIDC JWT documents. [STS Design section](STS-design-solution.md)
2.	STS Implementation: We will provide a detailed, step-by-step guide for the selected solution (using an API Gateway) and show how the Infrastructure as Code (IAC) uses this approach to enable STS during cluster deployment. [STS implementation section](STS-implementation.md)

