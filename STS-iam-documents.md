---
`cat openshift-deployer-role.json`

```
   {                                                                                       [7/9555]
    "Role"
        "Path": "/",
        "RoleName": "openshift-deployer-role",
        "RoleId": "{{ openshift_deployer_role_id }}",
        "Arn": "arn:{{ aws_partition }}:iam::{{ aws_account_id }}:role/openshift-deployer-role",
        "AssumeRolePolicyDocument": {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Principal": {
                        "AWS": [
                            "arn:{{ aws_partition }}:iam::{{ aws_account_id }}:role/openshift-deployer-role",
                            "arn:{{ aws_partition }}:iam::{{ aws_account_id }}:role/ec2-ssm-role",
                            "arn:{{ aws_partition }}:iam::{{ aws_account_id }}:user/openshift"
                        ],
                        "Service": "ec2.amazonaws.com"
                    },
                    "Action": [
                        "sts:AssumeRole",
                        "sts:SetSourceIdentity"
                    ]
                }
            ]
        },
        "Description": "Allows EC2 instances to call AWS services on your behalf.",
        "MaxSessionDuration": 3600
    }
}
```
`cat openshift-deployer-role-trust-relationship.json`

```
  {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": [
                    "ec2.amazonaws.com"
                ],
                "AWS": [
                    "arn:{{ aws_partition }}:iam::{{ aws_account_id }}:user/openshift",
                    "arn:{{ aws_partition }}:iam::{{ aws_account_id }}:user/openshiftt-deployer-role",
	  	    "arn:{{ aws_partition }}:iam::{{ aws_account_id }}:role/ec2-ssm-role"
                ]
            },
            "Action": [
                "sts:AssumeRole",
                "sts:SetSourceIdentity"
            ]
        }
    ]
}
```

`cat openshift-apigateway-iam-policy.json`

```
  {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "APIGatewayManagement",
            "Effect": "Allow",
            "Action": [
                "apigateway:POST",
                "apigateway:GET",
                "apigateway:PUT",
                "apigateway:PATCH",
                "apigateway:DELETE"
            ],
            "Resource": [
                "arn:{{ aws_partition }}:apigateway:{{ aws_region }}::/restapis",
                "arn:{{ aws_partition }}:apigateway:{{ aws_region }}::/restapis/*",
				"arn:{{ aws_partition }}:apigateway:{{ aws_region }}::/apis",
                "arn:{{ aws_partition }}:apigateway:{{ aws_region }}::/apis/*"
            ]
        },
        {
            "Sid": "APIGatewayTagging",
            "Effect": "Allow",
            "Action": [
                "apigateway:GET",
                "apigateway:PUT",
                "apigateway:DELETE"
            ],
            "Resource": [
                "arn:{{ aws_partition }}:apigateway:{{ aws_region }}::/tags/*"
            ]
        },
        {
            "Sid": "PassRoleToAPIGateway",
            "Effect": "Allow",
            "Action": "iam:PassRole",
            "Resource": "arn:{{ aws_partition }}:iam::{{ aws_account_id }}:role/*",
            "Condition": {
                "StringEquals": {
                    "iam:PassedToService": "apigateway.amazonaws.com"
                }
            }
        },
        {
            "Sid": "AllowExecuteOnAPIGateway",
            "Effect": "Allow",
            "Action": "execute-api:Invoke",
            "Resource": "arn:{{ aws_partition }}:execute-api:{{ aws_region }}:{{ aws_account_id }}:/*/*/*"
        },
        {
            "Sid": "AllowClientCertPermsOnAPIGateway",
            "Effect": "Allow",
            "Action": [
                "apigateway:POST",
                "apigateway:GET",
                "apigateway:PUT",
                "apigateway:DELETE"
            ],
            "Resource": [
                "arn:{{ aws_partition }}:apigateway:{{ aws_region }}::/clientcertificates",
                "arn:{{ aws_partition }}:apigateway:{{ aws_region }}::/clientcertificates/*"
            ]
        }
    ]
}
```

`cat openshift-sts-iam-policy.json`

```
  {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "OpenshiftSTSPerms",
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeInstances",
                "ec2:DescribeInstanceAttribute",
                "ec2:DescribeInstanceStatus",
                "ec2:DescribeTags",
                "ec2:DescribeSecurityGroups",
                "ec2:DescribeVpcs",
                "ec2:DescribeSubnets",
                "sts:GetCallerIdentity",
                "iam:CreateOpenIDConnectProvider",
                "iam:CreateRole",
                "iam:DeleteOpenIDConnectProvider",
                "iam:DeleteRole",
                "iam:DeleteRolePolicy",
                "iam:GetOpenIDConnectProvider",
                "iam:GetRole",
                "iam:GetUser",
                "iam:ListOpenIDConnectProviders",
                "iam:ListRolePolicies",
                "iam:ListRoles",
                "iam:PutRolePolicy",
                "iam:TagOpenIDConnectProvider",
                "iam:TagRole",
                "s3:CreateBucket",
                "s3:DeleteBucket",
                "s3:DeleteObject",
                "s3:GetBucketAcl",
                "s3:GetBucketTagging",
                "s3:GetObject",
                "s3:GetObjectAcl",
                "s3:GetObjectTagging",
                "s3:ListBucket",
                "s3:PutBucketAcl",
                "s3:PutBucketPolicy",
                "s3:GetBucketPolicy",
                "s3:PutBucketPublicAccessBlock",
                "s3:GetBucketPublicAccessBlock",
                "s3:PutBucketTagging",
                "s3:PutObject",
                "s3:PutObjectAcl",
                "s3:PutObjectTagging"
            ],
            "Resource": "*"
        }
    ]
}
```
`cat ssm-iam-policy.json`

```
  {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ssm:DescribeAssociation",
                "ssm:GetDeployablePatchSnapshotForInstance",
                "ssm:GetDocument",
                "ssm:DescribeDocument",
                "ssm:GetManifest",
                "ssm:GetParameter",
                "ssm:GetParameters",
                "ssm:ListAssociations",
                "ssm:ListInstanceAssociations",
                "ssm:PutInventory",
                "ssm:PutComplianceItems",
                "ssm:PutConfigurePackageResult",
                "ssm:UpdateAssociationStatus",
                "ssm:UpdateInstanceAssociationStatus",
                "ssm:UpdateInstanceInformation"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "ssmmessages:CreateControlChannel",
                "ssmmessages:CreateDataChannel",
                "ssmmessages:OpenControlChannel",
                "ssmmessages:OpenDataChannel"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "ec2messages:AcknowledgeMessage",
                "ec2messages:DeleteMessage",
                "ec2messages:FailMessage",
                "ec2messages:GetEndpoint",
                "ec2messages:GetMessages",
                "ec2messages:SendReply"
            ],
            "Resource": "*"
        }
    ]
}
```
`cat *-openshift-cloud-credential-operator-iam-role-trust-relationship.json`

```
  {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:{{ aws_partition }}:iam::{{ aws_account_id }}:oidc-provider/{{ aws_sts_apigateway_id }}.execute-api.{{ aws_region }}.amazonaws.com/{{ aws_deploy_stage }}"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "{{ aws_sts_apigateway_id }}.execute-api.{{ aws_region }}.amazonaws.com/{{ aws_deploy_stage }}:sub": "system:serviceaccount:openshift-cloud-credential-operator:cloud-credential-operator"
                }
            }
        }
    ]
}
```
`cat *-openshift-cloud-credential-operator-iam-policy.json`

```
  {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "iam:GetUser",
                "iam:GetUserPolicy",
                "iam:ListAccessKeys"
            ],
            "Resource": "*"
        }
    ]
}
```

`cat *-openshift-machine-api*-iam-role-trust-relationship.json`
```
  {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:{{ aws_partition }}:iam::{{ aws_account_id }}:oidc-provider/{{ aws_sts_apigateway_id }}.execute-api.{{ aws_region }}.amazonaws.com/{{ aws_deploy_stage }}"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "{{ aws_sts_apigateway_id }}.execute-api.{{ aws_region }}.amazonaws.com/{{ aws_deploy_stage }}:sub": "system:serviceaccount:openshift-machine-api:machine-api-controllers"
                }
            }
        }
    ]
}
```
  {
`cat *-openshift-machine-api*-iam-policy.json`
```
  {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:CreateTags",
                "ec2:DescribeAvailabilityZones",
                "ec2:DescribeDhcpOptions",
                "ec2:DescribeImages",
                "ec2:DescribeInstances",
                "ec2:DescribeInstanceTypes",
                "ec2:DescribeInternetGateways",
                "ec2:DescribeSecurityGroups",
                "ec2:DescribeRegions",
                "ec2:DescribeSubnets",
                "ec2:DescribeVpcs",
                "ec2:RunInstances",
                "ec2:TerminateInstances",
                "elasticloadbalancing:DescribeLoadBalancers",
                "elasticloadbalancing:DescribeTargetGroups",
                "elasticloadbalancing:DescribeTargetHealth",
                "elasticloadbalancing:RegisterInstancesWithLoadBalancer",
                "elasticloadbalancing:RegisterTargets",
                "elasticloadbalancing:DeregisterTargets",
                "iam:PassRole",
                "iam:CreateServiceLinkedRole"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "kms:ReEncrypt*",
                "kms:Decrypt",
                "kms:Encrypt",
                "kms:GenerateDataKey",
                "kms:GenerateDataKeyWithoutPlainText",
                "kms:DescribeKey"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "kms:RevokeGrant",
                "kms:CreateGrant",
                "kms:ListGrants"
            ],
            "Resource": "*",
            "Condition": {
                "Bool": {
                    "kms:GrantIsForAWSResource": true
                }
            }
        }
    ]
}
```
`cat *-openshift-ingress-operator*-iam-role-trust-relationship.json`
```
  {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:{{ aws_partition }}:iam::{{ aws_account_id }}:oidc-provider/{{ aws_sts_apigateway_id }}.execute-api.{{ aws_region }}.amazonaws.com/{{ aws_deploy_stage }}"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "{{ aws_sts_apigateway_id }}.execute-api.{{ aws_region }}.amazonaws.com/{{ aws_deploy_stage }}:sub": "system:serviceaccount:openshift-ingress-operator:ingress-operator"
                }
            }
        }
    ]
}
```
`cat *-openshift-ingress-operator*-iam-policy.json`
```
  {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "elasticloadbalancing:DescribeLoadBalancers",
                "route53:ListHostedZones",
                "route53:ListTagsForResources",
                "route53:ChangeResourceRecordSets",
                "tag:GetResources",
                "sts:AssumeRole"
            ],
            "Resource": "*"
        }
    ]
}
```
`cat *-openshift-cluster-cs-driver*-iam-role-relationship.json`
```
   {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:{{ aws_partition }}:iam::{{ aws_account_id }}:oidc-provider/{{ aws_sts_apigateway_id }}.execute-api.{{ aws_region }}.amazonaws.com/{{ aws_deploy_stage }}"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "{{ aws_sts_apigateway_id }}.execute-api.{{ aws_region }}.amazonaws.com/{{ aws_deploy_stage }}:sub": [
                        "system:serviceaccount:openshift-cluster-csi-drivers:aws-ebs-csi-driver-operator",
                        "system:serviceaccount:openshift-cluster-csi-drivers:aws-ebs-csi-driver-controller-sa"
                    ]
                }
            }
        }
    ]
}
```
`cat *-openshift-cluster-cs-driver*-iam-policy.json`
```
  {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:AttachVolume",
                "ec2:CreateSnapshot",
                "ec2:CreateTags",
                "ec2:CreateVolume",
                "ec2:DeleteSnapshot",
                "ec2:DeleteTags",
                "ec2:DeleteVolume",
                "ec2:DescribeInstances",
                "ec2:DescribeSnapshots",
                "ec2:DescribeTags",
                "ec2:DescribeVolumes",
                "ec2:DescribeVolumesModifications",
                "ec2:DetachVolume",
                "ec2:ModifyVolume",
                "ec2:DescribeAvailabilityZones",
                "ec2:EnableFastSnapshotRestores"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "kms:ReEncrypt*",
                "kms:Decrypt",
                "kms:Encrypt",
                "kms:GenerateDataKey",
                "kms:GenerateDataKeyWithoutPlainText",
                "kms:DescribeKey"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "kms:RevokeGrant",
                "kms:CreateGrant",
                "kms:ListGrants"
            ],
            "Resource": "*",
            "Condition": {
                "Bool": {
                    "kms:GrantIsForAWSResource": true
                }
            }
        }
    ]
}
```
`cat *-openshift-cloud-network*-iam-role-trust-relationship.json`
```
   {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:{{ aws_partition }}:iam::{{ aws_account_id }}:oidc-provider/{{ aws_sts_apigateway_id }}.execute-api.{{ aws_region }}.amazonaws.com/{{ aws_deploy_stage }}"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "{{ aws_sts_apigateway_id }}.execute-api.{{ aws_region }}.amazonaws.com/{{ aws_deploy_stage }}:sub": "system:serviceaccount:openshift-cloud-network-config-controller:cloud-network-config-controller"
                }
            }
        }
    ]
}
```
`cat *-openshift-cloud-network*-iam-policy.json`
```
  {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeInstances",
                "ec2:DescribeInstanceStatus",
                "ec2:DescribeInstanceTypes",
                "ec2:UnassignPrivateIpAddresses",
                "ec2:AssignPrivateIpAddresses",
                "ec2:UnassignIpv6Addresses",
                "ec2:AssignIpv6Addresses",
                "ec2:DescribeSubnets",
                "ec2:DescribeNetworkInterfaces"
            ],
            "Resource": "*"
        }
    ]
}
```
`cat *-openshift-image-registry-installer*-iam-role-trust-relationship.json`
```
  {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:{{ aws_partition }}:iam::{{ aws_account_id }}:oidc-provider/{{ aws_sts_apigateway_id }}.execute-api.{{ aws_region }}.amazonaws.com/{{ aws_deploy_stage }}"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "{{ aws_sts_apigateway_id }}.execute-api.{{ aws_region }}.amazonaws.com/{{ aws_deploy_stage }}:sub": [
                        "system:serviceaccount:openshift-image-registry:cluster-image-registry-operator",
                        "system:serviceaccount:openshift-image-registry:registry"
                    ]
                }
            }
        }
    ]
} 
```
`cat *-openshift-image-registry-installer*-iam-policy.json`
```
  {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:CreateBucket",
                "s3:DeleteBucket",
                "s3:PutBucketTagging",
                "s3:GetBucketTagging",
                "s3:PutBucketPublicAccessBlock",
                "s3:GetBucketPublicAccessBlock",
                "s3:PutEncryptionConfiguration",
                "s3:GetEncryptionConfiguration",
                "s3:PutLifecycleConfiguration",
                "s3:GetLifecycleConfiguration",
                "s3:GetBucketLocation",
                "s3:ListBucket",
                "s3:GetObject",
                "s3:PutObject",
                "s3:DeleteObject",
                "s3:ListBucketMultipartUploads",
                "s3:AbortMultipartUpload",
                "s3:ListMultipartUploadParts"
            ],
            "Resource": "*"
        }
    ]
}
```
