# Amazon Elasticsearch Service Access Control<a name="es-ac"></a>

Amazon Elasticsearch Service offers several ways of controlling access to your domains\. This section covers the various policy types, how they interact with each other, and how to create your own, custom policies\.

**Important**  
VPC support introduces some additional considerations to Amazon ES access control\. For more information, see [[ERROR] BAD/MISSING LINK TEXT](es-vpc.md#es-vpc-security)\.

## Types of Policies<a name="es-ac-types"></a>

Amazon ES supports three types of access policies:

+ [[ERROR] BAD/MISSING LINK TEXT](#es-ac-types-resource)

+ [[ERROR] BAD/MISSING LINK TEXT](#es-ac-types-identity)

+ [[ERROR] BAD/MISSING LINK TEXT](#es-ac-types-ip)

### Resource\-based Policies<a name="es-ac-types-resource"></a>

You attach resource\-based policies to domains\. These policies specify which actions a principal can perform on the domain's *subresources*\. Subresources include Elasticsearch indices and APIs\.

The [http://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_principal.html](http://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_principal.html) element specifies the accounts, users, or roles that are allowed access\. The [http://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_resource.html](http://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_resource.html) element specifies which subresources these principals can access\. The following resource\-based policy grants `test-user` full access \(`es:*`\) to `test-domain`:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": [
          "arn:aws:iam::123456789012:user/test-user"
        ]
      },
      "Action": [
        "es:*"
      ],
      "Resource": "arn:aws:es:us-west-1:987654321098:domain/test-domain/*"
    }
  ]
}
```

Two important considerations apply to this policy:

+ These privileges apply only to this domain\. Unless you create additional policies, `test-user` can't access other domains or even view a list of them in the Amazon ES dashboard\.

+ The trailing `/*` in the `Resource` element is significant\. Despite having full access, `test-user` can perform these actions only on the domain's subresources, not on the domain's configuration\.

  For example, `test-user` can make requests against an index \(`GET https://search-test-domain.us-west-1.es.amazonaws.com/test-index`\), but can't update the domain's configuration \(`POST https://es.us-west-1.amazonaws.com/2015-01-01/es/domain/test-domain/config`\)\. Note the difference between the two endpoints\. Accessing the configuration API requires an identity\-based policy\.

To further restrict `test-user`, you can apply the following policy:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": [
          "arn:aws:iam::123456789012:user/test-user"
        ]
      },
      "Action": [
        "es:ESHttpGet"
      ],
      "Resource": "arn:aws:es:us-west-1:987654321098:domain/test-domain/test-index/_search"
    }
  ]
}
```

Now `test-user` can perform only one operation: searches against `test-index`\. All other indices within the domain are inaccessible, and without permissions to use the `es:ESHttpPut` or `es:ESHttpPost` actions, `test-user` can't add or modify documents\.

Next, you might decide to configure a role for power users\. This policy allows `power-user-role` access to all HTTP methods, except for the ability to delete a critical index and its documents:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": [
          "arn:aws:iam::123456789012:role/power-user-role"
        ]
      },
      "Action": [
        "es:ESHttpDelete",
        "es:ESHttpGet",
        "es:ESHttpHead",
        "es:ESHttpPost",
        "es:ESHttpPut"
      ],
      "Resource": "arn:aws:es:us-west-1:987654321098:domain/test-domain/*"
    },
    {
      "Effect": "Deny",
      "Principal": {
        "AWS": [
          "arn:aws:iam::123456789012:role/power-user-role"
        ]
      },
      "Action": [
        "es:ESHttpDelete"
      ],
      "Resource": "arn:aws:es:us-west-1:987654321098:domain/test-domain/critical-index*"
    }
  ]
}
```

For information about all available actions, see [[ERROR] BAD/MISSING LINK TEXT](#es-ac-reference)\.

### Identity\-based policies<a name="es-ac-types-identity"></a>

Unlike resource\-based policies, which you attach to domains in Amazon ES, you attach identity\-based policies to users or roles using the AWS Identity and Access Management \(IAM\) service\. Just like resource\-based policies, identity\-based policies specify who can access a service, which actions they can perform, and if applicable, the resources on which they can perform those actions\.

While they certainly don't have to be, identify\-based policies tend to be more generic\. They often govern the basic, service\-level actions a user can perform\. After you have these policies in place, you can use resource\-based policies in Amazon ES to offer users additional permissions\.

Because identity\-based policies attach to users or roles \(principals\), the JSON doesn't specify a principal\. The following policy grants access to actions that begin with `Describe` and `List` and allows `GET` requests against all domains\. This combination of actions provides read\-only access:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": [
        "es:Describe*",
        "es:List*",
        "es:ESHttpGet"
      ],
      "Effect": "Allow",
      "Resource": "*"
    }
  ]
}
```

An administrator might have full access to Amazon ES:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": [
        "es:*"
      ],
      "Effect": "Allow",
      "Resource": "*"
    }
  ]
}
```

For more information about the differences between resource\-based and identity\-based policies, see [IAM Policies](http://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html) in the *IAM User Guide*\.

**Note**  
Users with the AWS managed `AmazonESReadOnlyAccess` policy can't see cluster health status in the console\. To allow them to see cluster health status, add the `"es:ESHttpGet"` action to an access policy and attach it to their accounts or roles\.

### IP\-based Policies<a name="es-ac-types-ip"></a>

IP\-based policies restrict access to a domain to one or more IP addresses or CIDR blocks\. Technically, IP\-based policies are not a distinct type of policy\. Instead, they are just resource\-based policies that specify an anonymous principal and include a special [http://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_condition.html](http://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_condition.html) element\.

The primary appeal of IP\-based policies is that they allow unsigned requests to an Amazon ES domain, which lets you use clients like [curl](https://curl.haxx.se/) and [[ERROR] BAD/MISSING LINK TEXT](es-kibana.md#es-managedomains-kibana) or access the domain through a proxy server\. To learn more, see [[ERROR] BAD/MISSING LINK TEXT](es-kibana.md#es-kibana-proxy)\.

**Note**  
If you enabled VPC access for your domain, you can't configure an IP\-based policy\. Instead, you can use [security groups](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_SecurityGroups.html) to control which IP addresses can access the domain\. For more information, see [[ERROR] BAD/MISSING LINK TEXT](es-vpc.md#es-vpc-security)\.

The following IP\-based access policy grants all requests that originate from `12.345.678.901` access to `test-domain`:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "*"
      },
      "Action": [
        "es:*"
      ],
      "Condition": {
        "IpAddress": {
          "aws:SourceIp": [
            "12.345.678.901"
          ]
        }
      },
      "Resource": "arn:aws:es:us-west-1:987654321098:domain/test-domain/*"
    }
  ]
}
```

## Signing Amazon ES Requests<a name="es-managedomains-signing-service-requests"></a>

Even if you configure a completely open resource\-based access policy, *all* requests to the Amazon ES configuration API must be signed\. If your policies specify IAM users or roles, requests to the Elasticsearch APIs also must be signed\. The signing method differs by API:

+ To make calls to the Amazon ES configuration API, we recommend that you use one of the [AWS SDKs](https://aws.amazon.com/tools/#sdk)\. The SDKs greatly simplify the process and can save you a significant amount of time compared to creating and signing your own requests\.

+ To make calls to the Elasticsearch APIs, you must sign your own requests\. For sample code, see [[ERROR] BAD/MISSING LINK TEXT](es-indexing.md#es-indexing-programmatic)\.

To sign a request, you calculate a digital signature using a cryptographic hash function, which returns a hash value based on the input\. The input includes the text of your request and your secret access key\. The hash function returns a hash value that you include in the request as your signature\. The signature is part of the `Authorization` header of your request\.

After receiving your request, Amazon ES recalculates the signature using the same hash function and input that you used to sign the request\. If the resulting signature matches the signature in the request, Amazon ES processes the request\. Otherwise, Amazon ES rejects the request\.

Amazon ES supports authentication using AWS Signature Version 4\. For more information, see [Signature Version 4 Signing Process](http://docs.aws.amazon.com/general/latest/gr/signature-version-4.html)\.

**Note**  
The service ignores parameters passed in URLs for HTTP POST requests that are signed with Signature Version 4\.

## When Policies Collide<a name="es-ac-conflict"></a>

Complexities arise when policies disagree or make no explicit mention of a user\. [Understanding How IAM Works](http://docs.aws.amazon.com/IAM/latest/UserGuide/intro-structure.html#intro-structure-authorization) in the *IAM User Guide* provides a concise summary of policy evaluation logic:

+ By default, all requests are denied\.

+ An explicit allow overrides this default\.

+ An explicit deny overrides any allows\.

For example, if a resource\-based policy grants you access to a domain, but an identify\-based policy denies you access, you are denied access\. If an identity\-based policy grants access and a resource\-based policy does not specify whether or not you should have access, you are allowed access\. See the following table of intersecting policies for a full summary of outcomes\.


****  

|  | Allowed in Resource\-based Policy | Denied in Resource\-based Policy | Neither Allowed nor Denied in Resource\-based Policy | 
| --- | --- | --- | --- | 
| Allowed in Identity\-based Policy |  Allow  | Deny | Allow | 
| Denied in Identity\-based Policy | Deny | Deny | Deny | 
| Neither Allowed nor Denied in Identity\-based Policy | Allow | Deny | Deny | 

## Policy Element Reference<a name="es-ac-reference"></a>

Amazon ES supports all the policy elements that are documented in the [IAM Policy Elements Reference](http://docs.aws.amazon.com/IAM/latest/UserGuide/AccessPolicyLanguage_ElementDescriptions.html)\. The following table shows the most common elements\.


****  

| JSON Policy Element | Summary | 
| --- | --- | 
| Version | The current version of the policy language is `2012-10-17`\. All access policies should specify this value\. | 
| Effect | This element specifies whether the statement allows or denies access to the specified actions\. Valid values are `Allow` or `Deny`\. | 
| Principal |  This element specifies the AWS account or IAM user or role that is allowed or denied access to a resource and can take several forms: [\[See the AWS documentation website for more details\]](http://docs.aws.amazon.com/elasticsearch-service/latest/developerguide/es-ac.html) Specifying the `*` wildcard enables anonymous access to the domain, which we don't recommend unless you add an IP\-based condition\.  | 
| Action  | Amazon ES uses the following actions for HTTP methods:[\[See the AWS documentation website for more details\]](http://docs.aws.amazon.com/elasticsearch-service/latest/developerguide/es-ac.html)Amazon ES uses the following actions for the configuration API:[\[See the AWS documentation website for more details\]](http://docs.aws.amazon.com/elasticsearch-service/latest/developerguide/es-ac.html)  You can use wildcards to specify a subset of actions, such as `"Action":"es:*"` or `"Action":"es:Describe*"`\. Certain `es:` actions support resource\-level permissions\. For example, you can give a user permissions to delete one particular domain without giving that user permissions to delete *any* domain\. Other actions apply only to the service itself\. For example, `es:ListDomainNames` makes no sense in the context of a single domain and thus requires a wildcard\.  Resource\-based policies differ from resource\-level permissions\. Resource\-based policies are full JSON policies that attach to domains\. Resource\-level permissions let you restrict actions to particular domains or subresources\. In practice, you can think of resource\-level permissions as an optional part of a resource\- or identity\-based policy\. The following identity\-based policy lists all `es:` actions and groups them according to whether they apply to the domain subresources \(`test-domain/*`\), to the domain configuration \(`test-domain`\), or only to the service \(`*`\):

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "es:ESHttpDelete",
        "es:ESHttpGet",
        "es:ESHttpHead",
        "es:ESHttpPost",
        "es:ESHttpPut"
      ],
      "Resource": "arn:aws:es:us-west-1:987654321098:domain/test-domain/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "es:CreateElasticsearchDomain",
        "es:DeleteElasticsearchDomain",
        "es:DescribeElasticsearchDomain",
        "es:DescribeElasticsearchDomainConfig",
        "es:DescribeElasticsearchDomains",
        "es:UpdateElasticsearchDomainConfig"
      ],
      "Resource": "arn:aws:es:us-west-1:987654321098:domain/test-domain"
    },
    {
      "Effect": "Allow",
      "Action": [
        "es:AddTags",
        "es:DeleteElasticsearchServiceRole",
        "es:DescribeElasticsearchInstanceTypeLimits",
        "es:ListDomainNames",
        "es:ListElasticsearchInstanceTypes",
        "es:ListElasticsearchVersions",
        "es:ListTags",
        "es:RemoveTags"
      ],
      "Resource": "*"
    }
  ]
}
```  While resource\-level permissions for `es:CreateElasticsearchDomain` might seem unintuitive—after all, why give a user permissions to create a domain that already exists?—the use of a wildcard lets you enforce a simple naming scheme for your domains, such as `"Resource": "arn:aws:es:us-west-1:987654321098:domain/my-team-name-*"`\.  Of course, nothing prevents you from including actions alongside less restrictive resource elements, such as the following:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "es:ESHttpGet",
        "es:DescribeElasticsearchDomain"
      ],
      "Resource": "*"
    }
  ]
}
``` To learn more about pairing actions and resources, see the `Resource` element in this table\. | 
| Condition |  Amazon ES supports all the conditions that are described in [Available Global Condition Keys](http://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_condition-keys.html#AvailableKeys) in the *IAM User Guide*\. When configuring an IP\-based policy, you specify the IP addresses or CIDR block as a condition, such as the following: 

```
"Condition": {
  "IpAddress": {
    "aws:SourceIp": [
      "192.0.2.0/32"
    ]
  }
}
```  | 
| Resource |  Amazon ES uses `Resource` elements in three basic ways: [\[See the AWS documentation website for more details\]](http://docs.aws.amazon.com/elasticsearch-service/latest/developerguide/es-ac.html) For details about which actions support resource\-level permissions, see the `Action` element in this table\.  | 

## Advanced Options and API Considerations<a name="es-ac-advanced"></a>

Amazon ES has several advanced options, one of which has access control implications: `rest.action.multi.allow_explicit_index`\. At its default setting of true, it allows users to bypass subresource permissions under certain circumstances\.

For example, consider the following resource\-based policy:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": [
          "arn:aws:iam::123456789012:user/test-user"
        ]
      },
      "Action": [
        "es:ESHttp*"
      ],
      "Resource": [
        "arn:aws:es:us-west-1:987654321098:domain/test-domain/test-index/*",
        "arn:aws:es:us-west-1:987654321098:domain/test-domain/_bulk"
      ]
    },
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": [
          "arn:aws:iam::123456789012:user/test-user"
        ]
      },
      "Action": [
        "es:ESHttpGet"
      ],
      "Resource": "arn:aws:es:us-west-1:987654321098:domain/test-domain/restricted-index/*"
    }
  ]
}
```

This policy grants `test-user` full access to `test-index` and the Elasticsearch [bulk](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-bulk.html) API\. It also allows `GET` requests to `restricted-index`\.

The following indexing request, as you might expect, fails due to a permissions error:

```
PUT https://search-test-domain.us-west-1.es.amazonaws.com/restricted-index/movie/1
{
  "title": "Your Name",
  "director": "Makoto Shinkai",
  "year": "2016"
}
```

Unlike the [index](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-index_.html) API, the bulk API lets you create, update, and delete many documents in a single call\. You often specify these operations in the request body, however, rather than in the request URL\. Because Amazon ES uses URLs to control access to domain subresources, `test-user` can, in fact, use the bulk API to make changes to `restricted-index`\. Even though the user lacks `POST` permissions on the index, the following request **succeeds**:

```
POST https://search-test-domain.us-west-1.es.amazonaws.com/_bulk
{ "index" : { "_index": "restricted-index", "_type" : "movie", "_id" : "1" } }
{ "title": "Your Name", "director": "Makoto Shinkai", "year": "2016" }
```

In this situation, the access policy fails to fulfill its intent\. To prevent users from bypassing these kinds of restrictions, you can change `rest.action.multi.allow_explicit_index` to false\. If this value is false, all calls to the bulk, [mget](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-multi-get.html), and [msearch](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-multi-search.html) APIs that specify index names in the request body stop working\. In other words, calls to `_bulk` no longer work, but calls to `test-index/_bulk` do\. This second endpoint contains an index name, so you don't need to specify one in the request body\.

Kibana relies heavily on mget and msearch, so it is unlikely to work properly after this change\. For partial remediation, you can leave `rest.action.multi.allow_explicit` as true and deny certain users access to one or more of these APIs\.

For information about changing this setting, see [[ERROR] BAD/MISSING LINK TEXT](es-createupdatedomains.md#es-createdomain-configure-advanced-options)\.

Similarly, the following resource\-based policy contains two subtle issues:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:user/test-user"
      },
      "Action": "es:ESHttp*",
      "Resource": "arn:aws:es:us-west-1:987654321098:domain/test-domain/*"
    },
    {
      "Effect": "Deny",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:user/test-user"
      },
      "Action": "es:*",
      "Resource": "arn:aws:es:us-west-1:987654321098:domain/test-domain/restricted-index/*"
    }
  ]
}
```

+ Despite the explicit deny, `test-user` can still make calls such as `GET https://search-test-domain.us-west-1.es.amazonaws.com/_all/_search` and `GET https://search-test-domain.us-west-1.es.amazonaws.com/*/_search` to access the documents in `restricted-index`\.

+ Because the `Resource` element references `restricted-index/*`, `test-user` doesn't have permissions to directly access the index's documents\. The user does, however, have permissions to *delete the entire index*\. To prevent access and deletion, the policy instead must specify `restricted-index*`\.

Rather than mixing broad allows and focused denies, the safest approach is to follow the principle of [least privilege](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html#grant-least-privilege) and grant only the permissions that are required to perform a task\.

## Configuring Access Policies<a name="es-ac-creating"></a>

+ For instructions on creating or modifying resource\- and IP\-based policies in Amazon ES, see [[ERROR] BAD/MISSING LINK TEXT](es-createupdatedomains.md#es-createdomain-configure-access-policies)\.

+ For instructions on creating or modifying identity\-based policies in IAM, see [Creating IAM Policies](http://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_create.html) in the *IAM User Guide*\.

## Additional Sample Policies<a name="es-ac-samples"></a>

Although this chapter includes many sample policies, AWS access control is a complex subject that is best understood through examples\. For more, see [Example Policies](http://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_examples.html) in the *IAM User Guide*\.