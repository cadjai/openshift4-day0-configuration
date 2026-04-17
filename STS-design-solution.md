# AWS STS OIDC Configuration within secure AWS private government regions Design 
The fundamental goal of this setup is to create an OIDC trust chain that allows OpenShift to federate credentials with AWS STS. This process hinges on AWS IAM's ability to access two small, non-sensitive files: the OIDC identity provider's public keys and its discovery metadata. Since this access must occur over HTTPS, the default S3 public access block in GovCloud and other secure regions presents a critical failure point.

To bridge this gap several alternatives will be presented and evaluated below to select the best option based on its availabilities on all fabrics.


## Solution Survey and Analysis

### OpenShift STS OIDC S3 Bucket Access in AWS GovCloud

#### Executive Summary

The goal here to **install and operate Red Hat OpenShift Container Platform (OCP) using AWS Security Token Service (STS) mode** in a secured GovCloud enclave as well as other AWS secure private governement regions, and the **OIDC identity provider configuration stored in an S3 bucket must be publicly readable via HTTPS** — which conflicts with the current GovCloud enclave's default policy of blocking S3 public access.

---

##### The Architecture: OpenShift with STS (Short-Lived Credentials)

When OpenShift is deployed on AWS using STS mode (also called "manual mode with STS" in OpenShift Container Platform), it replaces long-lived IAM user credentials with short-lived, automatically rotating STS tokens. This is the recommended security best practice for government environments and is essentially required for FedRAMP/FISMA compliance. Here's how it works:

1. **OpenShift pods need AWS permissions** (e.g., to manage EC2 instances, load balancers, EBS volumes, Route 53 records, EFS, etc.)

2. **Instead of embedding static IAM access keys**, OpenShift uses **IAM Roles for Service Accounts (IRSA)** — each OpenShift component gets its own IAM role with least-privilege permissions.

3. **The trust mechanism uses OIDC (OpenID Connect) federation**: Kubernetes/OpenShift issues JWT tokens (called Projected Service Account Tokens) to pods. AWS IAM validates these tokens against an OIDC identity provider to decide whether to grant STS temporary credentials.

4. **The OIDC discovery endpoint must be publicly accessible via HTTPS** — this is a hard requirement of the OpenID Connect specification (RFC 3986) and how AWS IAM validates the tokens. AWS IAM's STS service needs to reach the OIDC discovery document (`/.well-known/openid-configuration`) and the JSON Web Key Set (`keys.json`) to verify token signatures.

##### Where the S3 Bucket Fits In

The OpenShift Cloud Credential Operator utility (`ccoctl`) automates the STS setup. By default , it:

1. Creates an S3 bucket (e.g., `sts-test-cred-oidc`)
2. Uploads two critical files to the bucket:
   - `/.well-known/openid-configuration` — the OIDC discovery document
   - `/keys.json` — the JSON Web Key Set (JWKS) containing public keys
3. Makes the bucket publicly readable
4. Registers the S3 bucket URL as an IAM OIDC identity provider (e.g., `https://sts-test-cred-oidc.s3.us-gov-west-1.amazonaws.com`)

**The S3 bucket URL becomes the OIDC issuer URL that AWS IAM STS uses to validate tokens.**

##### The GovCloud and Secured GovCloud enclaves Conflict

Here is the core problem:

- **AWS IAM STS needs to read** `https://sts-test-cred-oidc.s3.us-gov-west-1.amazonaws.com/.well-known/openid-configuration` **over public HTTPS** to validate OIDC tokens
- **GovCloud accounts (and more secured  enclaves within GovCloud and other secure AWS government regions) typically have S3 Block Public Access enabled at the account level** (via Service Control Policies or account-level settings), which prevents any S3 bucket from being publicly accessible
- The `ccoctl` tool creates the bucket and tries to set it as public, but the account-level block overrides this
- Result: **AccessDenied** when anything (including AWS IAM itself) tries to reach the OIDC endpoint

##### The Error Explained

```
curl -vvv https://sts-test-cred-oidc.s3.us-gov-west-1.amazonaws.com/.well-known/openid-configuration
<Error><Code>AccessDenied</Code><Message>Access Denied</Message>...</Error>
```

This error confirms that the S3 bucket `sts-test-cred-oidc` exists but public read access is blocked. The tenant cannot make this bucket public because of account-level or organizational S3 Block Public Access policies — which is standard security posture in GovCloud environments.

**Important**: This is not just a tenant-facing issue. If AWS IAM STS cannot read these OIDC documents, the entire STS credential exchange chain breaks. Pods cannot assume IAM roles, and OpenShift operators cannot manage AWS resources.

---

#### Why "Pre-Signed URLs" Won't Work

Pre-signed URLs were initially considered as a possible solution, but this approach **will not work** for this use case because:

1. **Pre-signed URLs expire** — the OIDC endpoint needs to be permanently available
2. **AWS IAM STS is the client** — you cannot give AWS IAM a pre-signed URL as the OIDC provider endpoint; the IAM OIDC identity provider configuration requires a stable HTTPS URL
3. **The OIDC spec requires a standard HTTPS endpoint** — not a URL with query string authentication parameters

---

#### Solutions for GovCloud (and other secured private AWS government regions)

##### Solution 1: CloudFront Distribution as OIDC Endpoint (Implemented by OpenShift assuming it is available everywhere)

**The `ccoctl` tool now supports a `--create-private-s3-bucket` flag** that automates this approach. Instead of exposing the S3 bucket directly:

1. Keep the S3 bucket **private** (fully compliant with GovCloud S3 policies)
2. Create a **CloudFront distribution** that serves as the public HTTPS endpoint
3. CloudFront reads from the private S3 bucket using Origin Access Identity (OAI) or Origin Access Control (OAC)
4. Register the **CloudFront URL** (not the S3 URL) as the IAM OIDC identity provider

**GovCloud Complication**: CloudFront is **not available natively within GovCloud regions**. However, AWS documentation states you can create a CloudFront distribution in a standard AWS commercial region and point it at GovCloud S3 resources (treating GovCloud S3 as a custom/non-AWS origin). This requires:

- A linked commercial AWS account for CloudFront management
- Cross-boundary network connectivity (CloudFront to GovCloud S3)
- **Security consideration**: The OIDC discovery document and JWKS contain public keys and metadata only (no classified or CUI data), so serving them via CloudFront outside the GovCloud boundary may be acceptable. However, this needs to be validated with the agency's security team.


##### Solution 2: S3 VPC Endpoint + Bucket Policy (Internal-Only Access)

If the OIDC documents only need to be accessible from within the AWS network (not the public internet):

1. Create a **Gateway VPC Endpoint for S3** in the OpenShift VPC
2. Configure a **bucket policy** that allows access only from the VPC endpoint:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [{
       "Sid": "AllowVPCEndpointAccess",
       "Effect": "Allow",
       "Principal": "*",
       "Action": "s3:GetObject",
       "Resource": "arn:<aws_partition>:s3:::sts-test-cred-oidc/*",
       "Condition": {
         "StringEquals": {
           "aws:sourceVpce": "vpce-xxxxxxxxx"
         }
       }
     }]
   }
   ```
3. This does **NOT** require the S3 Block Public Access to be lifted — VPC endpoint-restricted access is not "public" access

**Critical Limitation**: This approach works for resources within the VPC but **AWS IAM STS itself may not access S3 through a VPC endpoint**. The IAM service needs to reach the OIDC endpoint, and IAM is a global/regional service that accesses S3 over the public internet, not through the customer's VPC. Therefore, **this solution alone is not sufficient** — It was tested and we confirmed that IAM STS Service was not able to access the URL of the OIDC provider documents.

##### Solution 3: API Gateway as OIDC Proxy

Deploy a regional and private API Gateway that:

1. Has a public HTTPS endpoint (or internal if agency network provides routing)
2. Proxies requests to the private S3 bucket
3. Serves the OIDC discovery and JWKS documents
4. Register this endpoint URL as the IAM OIDC identity provider

This keeps the S3 bucket private while providing the required public HTTPS endpoint.

##### Solution 4: ALB as OIDC Proxy

Deploy an Application Load Balancer that:

1. Has a public HTTPS endpoint (or internal if agency network provides routing)
2. Proxies requests to the private S3 bucket
3. Serves the OIDC discovery and JWKS documents
4. Register this endpoint URL as the IAM OIDC identity provider

This keeps the S3 bucket private while providing the required public HTTPS endpoint.

**Critical Limitation**: This approach by itself is not sufficient and might need to be combined with something like a VPC Endpoint or a web application serving the OIDC documents. It will ony work if the URL of the ALB is accessible to both the application and the AWS IAM STS Service. The IAM service needs to reach the OIDC endpoint, and IAM is a global/regional service that accesses S3 over the public internet, not through the customer's VPC or custom DNS. Therefore, **this solution alone is not sufficient** — It was tested and we confirmed that IAM STS Service was not able to access the URL of the OIDC provider documents if a custom DNS is used but might worn the internal DNS assigned to the ALB is used as long as the certificates associated with the ALB matches the same DNS.

##### Solution 5: Lambda@Edge or Lambda Function URL

Create a Lambda function that:

1. Reads the OIDC configuration and JWKS from the private S3 bucket
2. Serves them via a Lambda Function URL (public HTTPS endpoint)
3. Register the Lambda Function URL as the OIDC identity provider

This avoids CloudFront entirely and works within GovCloud (Lambda is available in GovCloud).

##### Solution 6: Request S3 Block Public Access Exception

Work with the AWS Cloud Services Broker (you) to:

1. Create an exception to the account-level S3 Block Public Access policy **for this specific bucket only**
2. The bucket would contain only the OIDC discovery document and JWKS (public keys) — no sensitive data
3. Apply a restrictive bucket policy that allows only `s3:GetObject` on the two specific keys

This is the simplest technical solution but may face governance/compliance objections.

##### Solution 7: Host OIDC Documents on an Internal Web Server

If the agency network provides routing between AWS IAM and internal resources:

1. Host the OIDC documents on an internal HTTPS web server (e.g., nginx on EC2)
2. Ensure it has a valid TLS certificate
3. Register this server's URL as the IAM OIDC identity provider

This keeps everything within the GovCloud boundary but requires managing additional infrastructure.

---

#### Selected Approach 

Given the tight security of the GovCloud enclave with strict S3 public access controls, and also given all of the organizational policies that will need to be changed to implement any of the options listed above, the path of least resistance seems to be either using the APIGatway option if it is available in all targeted regions,  use lambda function, or as a solution of last resort host a static web server that will serve the OIDC documents via https as long as the pki certificate used and the DNS match and are from AWS internal service so that both the application and the AWS STS Service can access the hosted web service.

After validation of several approaches we selected the APIGateway as the most promising solution. 
So the setup will look like OpenShift Pod -> AWS STS -> Regional API Gateway -> Private S3 Bucket

Why use a Regional API Gateway instead of a Private API Gateway?
The challenge is that the private API Gateway is only accessible from within the VPC via a VPC Endpoint. That works for any workload or any tests done from within the VPC. However, that will not work for the AWS Security Token Service (STS) or any other public AWS service because there exist outside of the VPC we are running in. 
Therefore the requirement to expose the OIDC discovery documents via a publicly resolvable and accessible HTTPS endpoint is most relevant for those public AWS services and therefore to satisfy it we need to use a publicly accessible API Gateway, which in our context will be a regional API Gateway. Doing so will allow the AWS STS service to review and approve token requests submitted by OpenShift workloads needing STS tokens. 

However, as already alluded to, it is important to also make sure the configuration applied to the API Gateway is valid and allows the AWS services access to the documents. To facilitate that testing, it is also recommended that a private API Gateway be created and setup for testing purpose and for any resources that might need access from within the VCP . The flow will then look like below with step 2. being optional.   

1.  OpenShift Pod -> AWS STS -> Regional API Gateway -> Private S3 Bucket  (required)

2.  OpenShift Pod -> AWS STS -> Private API Gateway -> Private S3 Bucket   (optional)


---

#### Reference Architecture Diagram

```
┌───────────────────────────────────────────────────────────────────────┐
│  OpenShift Container Platform Cluster                                 |
|       (GovCloud - us-gov-west-1 or any secure private                 | 
|             government regions)                                       │
│                                                                       │
│                                                                       │
│  ┌──────────┐   JWT Token    ┌──────────────────┐                     │
│  │ OCP Pod  │ ──────────────>│  AWS STS         │                     │
│  │ (with SA)│                │  AssumeRoleWith  │                     │
│  └──────────┘                │  WebIdentity     │                     │
│                              └────────┬─────────┘                     │
│                                       │                               │
│                          "Validate    │                               │
│                           JWT against │                               │
│                           OIDC"       │                               │
│                                       ▼                               │
│                           ┌──────────────────────┐                    │
│                           │    IAM OIDC          │                    │
│                           │    Identity          │                    │
│                           │    Provider          │                    │
│                           └──────────┬───────────┘                    │
│                                      │                                │
│                         "Fetch JWKS  │                                │
│                          from OIDC   │                                │
│                          endpoint"   │                                │
│                                      ▼                                │
│                     ┌──────────────────────────────────┐              │
│                     │  OIDC HTTPS Endpoint             │              │
│                     │  (MUST be publicly reachable     │              │
│                     │   by AWS IAM service)            │              │
│                     │                                  │              │
│                     │  OPTIONS:                        │              │
│                     │  A) Public S3 ← BLOCKED          │              │
│                     │  B) Regional API GW → Private S3 │              │
│                     │  C) Lambda Function URL → S3     │              │
│                     │  D) ALB/API GW → S3              │              │
│                     │  E) ALB/VPCE → S3                │              │
│                     │  F) ALB/Static Web Proxy → S3    │              │
│                     │  G) Request UNBLOCKED Public S3  │              │
│                     └──────────────┬───────────────────┘              │
│                                    │                                  │
│                                    ▼                                  │
│                     ┌───────────────────────────────┐                 │
│                     │  Private S3 Bucket            │                 │
│                     │  sts-test-cred-oidc           │                 │
│                     │                               │                 │
│                     │  /.well-known/                │                 │
│                     │    openid-configuration       │                 │
│                     │  /keys.json                   │                 │
│                     └───────────────────────────────┘                 │
└───────────────────────────────────────────────────────────────────────┘
```

---

#### Summary

To enable secure STS credentials in OpenShift, a trust relationship with AWS IAM must be established. This requires AWS to access the cluster's OIDC public keys and metadata via a public HTTPS endpoint. In secure regions like GovCloud, default policies block public S3 access, breaking this trust chain.

After evaluating several alternatives, we selected AWS API Gateway as the best solution. It provides a secure HTTPS endpoint to expose the two necessary, non-sensitive files from a private S3 bucket, allowing AWS IAM to validate tokens and issue temporary credentials.

