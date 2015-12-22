﻿<properties
	pageTitle="Azure Resource Manager Policy | Microsoft Azure"
	description="Describes how to use Azure Resource Manager Policy to prevent violations at different scopes like subscription, resource groups or individual resources."
	services="azure-resource-manager"
	documentationCenter="na"
	authors="ravbhatnagar"
	manager="ryjones"
	editor=""/>

<tags
	ms.service="azure-resource-manager"
	ms.devlang="na"
	ms.topic="article"
	ms.tgt_pltfrm="na"
	ms.workload="na"
	ms.date="12/18/2015"
	ms.author="gauravbh;tomfitz"/>

# Use Policy to manage resources and control access

Azure Resource Manager now allows you to control access through custom
policies. With policies, you can prevent users in your organization from breaking conventions that are needed to manage your organization's resources. 

You create policy definitions that describe the actions or resources that are specifically denied. 
You assign those policy definitions at the desired scope, such as the subscription,
resource group, or an individual resource. 

In this article, we will explain the basic structure of the policy
definition language that you can use to create policies. Then we will
describe how you can apply these policies at different scopes and
finally we will show some examples of how you can achieve this through
REST API.

Policy is currently available as a preview.

## How is it different from RBAC?

There are a few key differences between policy and role-based access control, but the first thing to understand is that 
policies and RBAC work together. To be able to use policy, the user must be authenticated through RBAC. Unlike RBAC, policy is a 
default allow and explicit deny system. 

RBAC focuses on the actions a **user** can perform at different scopes. 
For example, a particular user is added to the contributor role for a resource group at the desired scope, so the user can make changes to that 
resource group. 

Policy focuses on **resource** actions at various scopes. For example, through policies, you can 
control the types of resources that can be provisioned or restrict the locations in which the 
resources can be provisioned.

## Common Scenarios

One common scenario is to require departmental tags for chargeback
purpose. An organization might want to allow operations only when the
appropriate cost center is associated; otherwise, they will deny the request.
This would help them charge the appropriate cost center for the
operations performed.

Another common scenario is that the organization might want to control
the locations where resources are created. Or they might want to control
access to the resources by allowing only certain types of resources to
be provisioned.

Similarly, an organization can control the service catalog or enforce
the desired naming conventions for the resources.

Using policies, these scenarios can easily be achieved as described below.

## Policy Definition structure

Policy definition is created using JSON. It consists of one or more
conditions/logical operators which define the actions and an effect
which tells what happens when the conditions are fulfilled.

Basically, a policy contains the following:

**Condition/Logical operators:** It contains a set of conditions which
can be manipulated through a set of logical operators.

**Effect:** This describes what the effect will be when the condition is
satisfied – either deny or audit. An audit effect will emit a warning
event service log. For example, an administrator can create a policy which causes an audit if anyone creates a large VM, then review the logs later.

    {
      "if" : {
        <condition> | <logical operator>
      },
      "then" : {
        "effect" : "deny | audit"
      }
    }


## Logical Operators

The supported logical operators along with the syntax are listed below:

| Operator Name		| Syntax		 |
| :------------- | :------------- |
| Not			 | "not" : {&lt;condition  or operator &gt;}			 |
| And			| "allOf" : [ {&lt;condition  or operator &gt;},{&lt;condition  or operator &gt;}] |
| Or						 | "anyOf" : [ {&lt;condition  or operator &gt;},{&lt;condition  or operator &gt;}] |

Resource Manager enables you to specify complex logic in your policy through nested operators. For example, you can deny resource creation in a particular location for a specified resource type. An example of nested 
operators is shown below.

## Conditions

A condition evaluates whether a **field** or **source** meets certain criteria. The supported condition names and syntax are listed below:

| Condition Name | Syntax				 |
| :------------- | :------------- |
| Equals			 | "equals" : "&lt;value&gt;"				|
| Like					| "like" : "&lt;value&gt;"					 |
| Contains			| "contains" : "&lt;value&gt;"|
| In						| "in" : [ "&lt;value1&gt;","&lt;value2&gt;" ]|
| ContainsKey	 | "containsKey" : "&lt;keyName&gt;" |

## Fields and Sources

Conditions are formed through the use of fields and sources. A field represents properties in the resource request payload. A source represents characteristics of the request itself. 

The following fields and sources are supported:

Fields: **name**, **kind**, **type**, **location**, **tags**, **tags.***.

Sources: **action**. 

To get more information about actions, see [RBAC - Built in Roles] (active-directory/role-based-access-built-in-roles.md). Currently, policy only works on PUT requests. 

## Policy Definition Examples

Now let's take a look at how we will define the policy to achieve the
scenarios listed above.

### Chargeback: Require departmental tags

The below policy denies all requests which don’t have a tag containing
"costCenter" key.

    {
      "if": {
        "not" : {
          "field" : "tags",
          "containsKey" : "costCenter"
        }
      },
      "then" : {
        "effect" : "deny"
      }
    }


### Geo Compliance: Ensure resource locations

The below example shows a policy which will deny all requests where location is not North Europe or West Europe.

    {
      "if" : {
        "not" : {
          "field" : "location",
          "in" : ["northeurope" , "westeurope"]
        }
      },
      "then" : {
        "effect" : "deny"
      }
    }

### Service Curation: Select the service catalog

The below example shows the use of source. It shows that actions only on
the services of type Microsoft.Resources/\*, Microsoft.Compute/\*,
Microsoft.Storage/\*, Microsoft.Network/\* are allowed. Anything else
will be denied.

    {
      "if" : {
        "not" : {
          "anyOf" : [
            {
              "source" : "action",
              "like" : "Microsoft.Resources/*"
            },
            {
              "source" : "action",
              "like" : "Microsoft.Compute/*"
            },
            {
              "source" : "action",
              "like" : "Microsoft.Storage/*"
            },
            {
              "source" : "action",
              "like" : "Microsoft.Network/*"
            }
          ]
        }
      },
      "then" : {
        "effect" : "deny"
      }
    }

### Naming Convention

The below example shows the use of wildcard which is supported by the condition
"like". The condition states that if the name does match the mentioned pattern (namePrefix\*nameSuffix) then deny
the request.

    {
      "if" : {
        "not" : {
          "field" : "name",
          "like" : "namePrefix*nameSuffix"
        }
      },
      "then" : {
        "effect" : "deny"
      }
    }
    
### Tag requirement just for Storage resources

The below example shows how to nest logical operators to require an application tag for only Storage resources.

    {
        "if": {
            "allOf": [
              {
                "not": {
                  "field": "tags",
                  "containsKey": "application"
                }
              },
              {
                "source": "action",
                "like": "Microsoft.Storage/*"
              }
            ]
        },
        "then": {
            "effect": "audit"
        }
    }

## Policy Assignment

Policies can be applied at different scopes like subscription, resource
groups and individual resources. Policies are inherited by all child
resources. So if a policy is applied to a resource group, it will be
applicable to all the resources in that resource group.

## Creating a Policy

This section provides detail on how a policy can be created using REST
API.

### Create Policy Definition with REST API

You can create a policy with the [REST API for Policy Definitions](https://msdn.microsoft.com/library/azure/mt588471.aspx). The REST API enables you to create and delete policy definitions, and get information about existing definitions.

To create a new policy, run:

    PUT https://management.azure.com/subscriptions/{subscription-id}/providers/Microsoft.authorization/policydefinitions/{policyDefinitionName}?api-version={api-version}

With a request body similar to the following:

    {
      "properties":{
        "policyType":"Custom",
        "description":"Test Policy",
        "policyRule":{
          "if" : {
            "not" : {
              "field" : "tags",
              "containsKey" : "costCenter"
            }
          },
          "then" : {
            "effect" : "deny"
          }
        }
      },
      "id":"/subscriptions/########-####-####-####-############/providers/Microsoft.Authorization/policyDefinitions/testdefinition",
      "type":"Microsoft.Authorization/policyDefinitions",
      "name":"testdefinition"
    }


The policy-definition can be defined as one of the examples shown above.
For api-version use *2015-10-01-preview*. For examples and more details,
see [REST API for Policy Definitions](https://msdn.microsoft.com/library/azure/mt588471.aspx).

### Create Policy Definition using PowerShell

You can create a new policy definition using the New-AzureRmPolicyDefinition cmdlet as shown below. The below examples creates a policy for allowing resources only in North Europe and West Europe.

    $policy = New-AzureRmPolicyDefinition -Name regionPolicyDefinition -Description "Policy to allow resource creation onlyin certain regions" -Policy '{	"if" : {
    	    			    "not" : {
    	      			    	"field" : "location",
    	      			    		"in" : ["northeurope" , "westeurope"]
    	    			    	}
    	    		          },
    	      		    		"then" : {
    	    			    		"effect" : "deny"
    	      			    		}
    	    		    	}'    		

The output of execution is stored in $policy object as it can used later during policy assignment. For the policy parameter, the path to a .json file containing the policy can also be provided instead of specifying the policy inline as shown below.

    New-AzureRmPolicyDefinition -Name regionPolicyDefinition -Description "Policy to allow resource creation only in certain 	regions" -Policy "path-to-policy-json-on-disk"


## Applying a Policy

### Policy Assignment with REST API

You can apply the policy definition at the desired scope through the [REST API for policy assignments](https://msdn.microsoft.com/library/azure/mt588466.aspx).
The REST API enables you to create and delete policy assignments, and get information about existing assignments.

To create a new policy assignment, run:

    PUT https://management.azure.com /subscriptions/{subscription-id}/providers/Microsoft.authorization/policyassignments/{policyAssignmentName}?api-version={api-version}

The {policy-assignment} is the name of the policy assignment. For
api-version use *2015-10-01-preview*. 

With a request body similar to the following:

    {
      "properties":{
        "displayName":"VM_Policy_Assignment",
        "policyDefinitionId":"/subscriptions/########/providers/Microsoft.Authorization/policyDefinitions/testdefinition",
        "scope":"/subscriptions/########-####-####-####-############"
      },
      "id":"/subscriptions/########-####-####-####-############/providers/Microsoft.Authorization/policyAssignments/VMPolicyAssignment",
      "type":"Microsoft.Authorization/policyAssignments",
      "name":"VMPolicyAssignment"
    }

For examples and more details, see [REST API for Policy Assignments](https://msdn.microsoft.com/library/azure/mt588466.aspx).

### Policy Assignment using PowerShell

You can apply the policy created above through PowerShell to the desired scope by using the New-AzureRmPolicyAssignment cmdlet as shown below:

    New-AzureRmPolicyAssignment -Name regionPolicyAssignment -PolicyDefinition $policy -Scope    /subscriptions/########-####-####-####-############/resourceGroups/<resource-group-name>
        
Here $policy is the policy object that was returned as a result of executing the New-AzureRmPolicyDefinition cmdlet as shown above. The scope here is the name of the resource group you specify.

If you want to remove the above policy assignment, you can do it as follows:

    Remove-AzureRmPolicyAssignment -Name regionPolicyAssignment -Scope /subscriptions/########-####-####-####-############/resourceGroups/<resource-group-name>

You can get, change or remove policy definitions through Get-AzureRmPolicyDefinition, Set-AzureRmPolicyDefinition and Remove-AzureRmPolicyDefinition cmdlets respectively.

Similarly, you can get, change or remove policy assignments through the Get-AzureRmPolicyAssignment, Set-AzureRmPolicyAssignment and Remove-AzureRmPolicyAssignment cmdlets respectively.

##Policy Audit Events

After you have applied your policy, you will begin to see policy-related events. You can either go to portal or use PowerShell to get this data. 

To view all events that related to deny effect, you can use the following command. 

    Get-AzureRmLog | where {$_.subStatus -eq "Forbidden"}     

To view all events related to audit effect, you can use the following command. 

    Get-AzureRmLog | where {$_.OperationName -eq "Microsoft.Authorization/policies/audit/action"} 
    

