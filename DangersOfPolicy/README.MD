# Avoiding Azure Policy Pitfalls: The Hidden Complexity of Managed Services

*The hidden complexity behind Azure Policy: why managed services change everything and how to implement governance that actually works in practice*

## Introduction

Azure Policy appears straightforward in basic scenarios--apply a rule, enforce governance, maintain compliance. But this simplicity is deceptive. When you introduce managed services like Azure Kubernetes Service (AKS), Azure Synapse, and Azure Databricks into the equation, Azure Policy reveals layers of complexity that can catch even experienced practitioners off guard.

The challenge isn't just about avoiding "careless" policy deployments. It's about understanding that managed services fundamentally change how Azure Policy works. These services automatically provision and manage underlying Azure resources in ways that can conflict with governance policies, creating architectural disconnects that are difficult to predict and troubleshoot.

In this article, we'll use AKS as a detailed case study to demonstrate why simple policy approaches break down with managed services. We'll start with a real-world scenario where a basic tagging policy silently breaks Kubernetes deployments, then progress through increasingly sophisticated solutions--from the tactical "Monitor First" approach for existing environments to custom policy definitions required for greenfield deployments.

This progression from portal-based governance to policy-as-code solutions reflects a broader reality: while Azure Policy can be managed simply for basic scenarios, enterprise governance with managed services typically requires Infrastructure as Code approaches and frameworks like Enterprise Policy as Code (EPAC).

After reading this article, you'll understand not just how to avoid policy-related service disruptions, but when and why Azure Policy complexity becomes unavoidable. You'll have practical methodologies for both brownfield and greenfield scenarios, and you'll recognize the signs that indicate when you've outgrown portal-based policy management.

### Who needs to understand this?

This article is essential if you:
- Are planning to implement Azure Policy in environments with managed services
- Have experienced unexplained failures after policy deployments
- Manage AKS, Synapse, Databricks, or other services that auto-provision resources
- Need to balance governance requirements with operational reliability
- Want to avoid costly downtime from policy-related service disruptions

## Policy Implementation: Understanding Your Environment

Before diving into implementation approaches, it's important to understand the two scenarios you might be working with:

**Brownfield Environments**: Existing Azure environments with established workloads and services. However, even in brownfield environments, new resources are constantly being created--whether by users deploying new workloads or by managed services provisioning infrastructure automatically.

**Greenfield Environments**: New Azure environments where you're starting fresh, typically wanting governance policies active from day one.

The key challenge is that both scenarios involve new resource creation, but they differ in your ability to assess policy impact beforehand.

**The Risky Approach: Enforce First**
- Quick to implement
- Instant enforcement
- BUT: Can cause unexpected service disruptions
- Hard to troubleshoot when things break

**The Safe Approach: Monitor First**
- Requires initial planning
- Delayed enforcement
- BUT: Avoids operational surprises for existing patterns
- Provides clear impact assessment
- BUT: May still encounter issues with new resource types or managed services

*Note: The "Monitor First" approach helps with known resource patterns but doesn't eliminate all risks. Advanced scenarios require custom policy definitions that can handle dynamic resource creation patterns.*

![Two Approaches](/DangersOfPolicy/img/1-twoapproaches-EN.png)

## The Core Problem: Why Managed Services Complicate Azure Policy

Azure managed services automatically provision and manage underlying Azure resources on your behalf. This creates a fundamental challenge when governance policies with restrictive effects are enforced without considering their operational requirements.

To illustrate this challenge, let's examine Azure Kubernetes Service (AKS) as a representative example. When you deploy workloads to AKS, the platform automatically provisions Azure resources like managed disks for persistent storage, public IPs for load balancers, and network security groups. AKS interacts with Azure Resource Manager to create these resources as needed, but Kubernetes has no native mechanisms to satisfy many Azure Policy requirements.

This architectural disconnect between service-level operations and Azure Policy requirements isn't unique to AKS. Many Azure managed services face the same challenge:

- **Azure Synapse**: Creates storage accounts, data lake storage, and analytics resources
- **Azure Databricks**: Provisions virtual networks, VMs, and storage  
- **Azure Machine Learning**: Manages compute instances and associated infrastructure
- **Logic Apps**: Creates connection resources and integration service environments
- **App Service**: Provisions storage, networking components, and certificates

Any policy that can block or modify resource creation may impact these services' ability to function properly. The following demonstration with AKS illustrates patterns that apply broadly across Azure's managed service ecosystem.

## The Test Environment: AKS as a Case Study

To demonstrate this issue, I set up a test focused on AKS, though the lessons apply broadly. This example represents a typical brownfield scenario where policies are introduced to an existing, functioning environment:

1. **Initial setup**: Deployed a functioning AKS cluster and validated basic functionality
2. **Introducing the policy**: Deployed a policy denying resource creation without a specific tag (as a representative example)
3. **The impact**: Observed how the policy affected basic Kubernetes operations

### Initial Setup

I deployed a standard AKS cluster and connected to it as the cluster administrator. To verify normal functionality, I deployed a simple WordPress application, which requires persistent storage and networking resources.

As expected, the application deployed successfully in the default configuration.

### Introducing the Policy

For this demonstration, I simulated a common governance requirement: enforcing cost tracking by requiring a "costcenter" tag on all Azure resources. I used the built-in policy "Require a tag on resources" with the DENY effect, which prevents the creation of any resource without the specified tag.

![Assign Policy](/DangersOfPolicy/img/2%20-%20Assign%20policy.png)

![Assign Policy](/DangersOfPolicy/img/3%20-%20Assign%20policy%202.png)

This example represents a broader class of policies that can restrict resource creation based on various governance requirements.

### The Impact

When I attempted to redeploy WordPress after applying the policy, the results revealed a critical issue:

1. No immediate error messages appeared at the Kubernetes level

![Consequence](/DangersOfPolicy/img/4%20-%20consequence1.png)

2. The pods and PersistentVolumeClaims (PVCs) remained stuck in "Pending" state

![Consequence](/DangersOfPolicy/img/5%20-%20consequence2.png)

3. The application failed to start, with no clear indication of the root cause from the Kubernetes perspective

Diving deeper into the logs revealed the actual issue: "RequestDisallowedByPolicy." When AKS attempted to create managed disks for the WordPress PVCs, Azure Resource Manager denied the operations due to policy violations.

![K8slogs](/DangersOfPolicy/img/6%20-%20Kubernetes%20logs.png)

Looking at the Activity Logs in the MC_* resource group confirmed numerous failed operations initiated by the AKS-managed identity, all blocked by the policy.

![Activity logs](/DangersOfPolicy/img/7%20-%20Azure%20activity%20logs.png)

![Activity logs](/DangersOfPolicy/img/8%20-%20Activity%20log%20entry.png)

The MC_* resource group is automatically created by Azure when you deploy an AKS cluster--it's where AKS stores all the underlying infrastructure components it manages on your behalf, including virtual machines, managed disks, network security groups, load balancers, and public IP addresses. 

This separation allows AKS to manage infrastructure without requiring users to have direct access to these underlying resources, but it also means that policy violations in this hidden layer can cause mysterious failures at the application level.

## The Fundamental Challenge

The core problem extends beyond just AKS and applies to many Azure managed services:

1. **Architectural Disconnect**: Service-specific APIs have no awareness of Azure Policy requirements
2. **Transparent Resource Creation**: Managed services create Azure resources without exposing all Azure-specific properties
3. **Limited Control**: Service interfaces provide no mechanism to specify many Azure-specific policy requirements
4. **Silent Failures**: Policy violations often manifest as generic failures with minimal diagnostics

![Architectural disconnect](/DangersOfPolicy/img/AKS%20Policy%20Disconnect.png)

When you deploy policies with restrictive effects like DENY, MODIFY, or even DEPLOYIFNOTEXISTS without proper preparation, you can create situations where managed services cannot fulfil their operational responsibilities.

## The hotfix: Policy Exemption

To resolve the immediate issue with AKS, I created a policy exemption for the MC_* resource group:

![Exemption](/DangersOfPolicy/img/9%20-%20Exemption%20Assignment.png)

After applying the exemption and redeploying WordPress, the application deployed successfully. The policy continued to apply to all other resources, but AKS could now provision the necessary infrastructure.

![Post-exemption Deploy](/DangersOfPolicy/img/10%20-%20DeployWPPostExemption.png)

Similar exemptions would be required for resource groups managed by other services like Synapse, Databricks, and others.

While exemptions provide a tactical solution, they represent a reactive approach. A more strategic methodology is needed.

## The Better Approach: Monitor First

> **The key principle for safely implementing Azure Policy with any managed service is: MONITOR FIRST.**

Before deploying any policy with potentially disruptive effects, always:

1. **Deploy in Audit Mode or disable enforcement**: Set the policy effect to AUDIT or disable policy enforcement
2. **Evaluate Impact**: Analyse which resources and operations would be affected before enforcing
3. **Plan Remediation or exemptions**: Determine how to address policy requirements without disrupting services
4. **Communicate**: Inform affected teams about upcoming policy enforcement
5. **Implement Gradually**: Apply enforcement only after proper preparation

This approach applies to all Azure managed services, not just AKS.

To demonstrate this approach with our AKS example, I:

1. Switched the policy enforcement from "Enabled" to "Disabled"

![PolicyEnforceDisabled](/DangersOfPolicy/img/11%20-%20DisablePolicyEnforcement.png)

2. Removed the exemption for the MC_* resource group
3. Redeployed WordPress, which worked just fine

![AuditDeploy](/DangersOfPolicy/img/12%20-%20DeployWPAuditmode.png)

4. Ran a policy compliance scan to identify affected resources

![Compliance Scan](/DangersOfPolicy/img/13%20-%20PolicyComplianceScan.png)

The scan revealed non-compliant resources in the MC_* resource group, providing valuable insights about the potential impact without causing disruption.

![Compliance Result](/DangersOfPolicy/img/14%20-%20Policy%20Compliance%20Result.png)

A proper follow-up will discover what these non-compliant resources are and how they are managed. In this case the logical conclusion is to exempt the MC_ resourcegroup from the policy assignment to let AKS manage its resources without interruption.

## Beyond AKS: Applying These Lessons Across Azure

While this article has focused on AKS as a case study, the same methodology should be applied when implementing policies that might affect:

- **Azure Synapse**: Check resource groups containing Synapse workspaces
- **Azure Databricks**: Review managed resource groups with naming patterns like "databricks-rg-*"
- **App Service**: Look for App Service plans and their supporting resources
- **Azure Machine Learning**: Examine ML workspace resource groups
- **Any PaaS offering**: Consider if and how the service provisions and manages resources

For each service, identify the resource groups and resources that are automatically managed, and evaluate policy impacts before enforcement.

## Advanced Strategies: Understanding Policy Rules and Greenfield Scenarios

The "Monitor First" approach works well for existing environments, but what if you're implementing governance in a greenfield scenario where you want DENY policies enforced from the start? To understand why this requires a different approach, let's examine how Azure Policy rules work.

### The Greenfield Challenge

In greenfield environments, you face a catch-22: you want restrictive policies active from day one, but you can't use the "Monitor First" approach to identify exemptions because there are no existing resources to evaluate.

For our AKS example, this creates a problem:
- You want the tagging policy to DENY resources without proper tags
- But AKS will create the MC_* resource group during deployment
- The policy will block this creation, causing deployment failure
- You can't exempt the resource group beforehand because it doesn't exist yet

### Portal Limitations and the Need for Custom Policies

The Azure portal has significant limitations for handling these scenarios:
- **No wildcard support in notScopes**: You can't exclude MC_* patterns in policy assignments through the portal
- **Static exemptions only**: Exemptions must target specific, existing resources
- **Limited conditional logic**: Complex exclusion patterns require custom policy definitions

### The Greenfield Reality: Pipeline Failures

To illustrate this challenge concretely, here's what happens when you try to deploy AKS in a greenfield environment with restrictive policies already active:

[Screenshot of pipeline failure showing policy blocking AKS deployment]

The deployment fails when AKS attempts to create the MC_* resource group and the required resources within. There's no good way to prevent this through portal-based policy management.

The pipeline error clearly shows the policy violation, but the timing makes it impossible to create exemptions reactively, as the scope where you'd want to create an exemption for does not yet exist.

This is where understanding how policy rules work becomes crucial.

## Comparing Built-in vs Custom Policy Rules

To understand why we need custom policies for greenfield scenarios, let's compare how the built-in and custom policy rules handle our AKS challenge.

### The Built-in Policy Rule

The built-in "Require a tag on resources" policy has a simple rule structure:

```json
"policyRule": {
    "if": {
        "field": "[concat('tags[', parameters('tagName'), ']')]",
        "exists": "false"
    },
    "then": {
        "effect": "deny"
    }
}
```

This rule asks a single question: "Does this resource have the required tag?" If not, deny it. It's simple but inflexible - there's no way to exclude managed service resource groups.

### The Custom Policy Rule: Adding Intelligence

The custom policy takes a more sophisticated approach with multiple conditions using allOf:
```json
"policyRule": {
    "if": {
        "allOf": [
            // Condition 1: Resource type filtering
            {
                "anyOf": [
                    {
                        "field": "type",
                        "in": "[parameters('resourceTypeList')]"
                    },
                    {
                        "allOf": [
                            {
                                "value": "[length(parameters('resourceTypeList'))]",
                                "equals": 0
                            },
                            {
                                "count": {
                                    "value": "[parameters('excludedResourceTypes')]",
                                    "name": "excludedResourceTypes",
                                    "where": {
                                        "field": "type",
                                        "like": "[current('excludedResourceTypes')]"
                                    }
                                },
                                "equals": 0
                            }
                        ]
                    }
                ]
            },
            // Condition 2: Tag requirement (same as built-in)
            {
                "field": "[concat('tags[', parameters('tagName'), ']')]",
                "exists": "false"
            },
            // Condition 3: Resource group exclusions
            {
                "count": {
                    "value": "[parameters('excludedRG')]",
                    "name": "excludedRG",
                    "where": {
                        "value": "[resourceGroup().name]",
                        "like": "[current('excludedRG')]"
                    }
                },
                "equals": 0
            }
        ]
    },
    "then": {
        "effect": "[parameters('effect')]"
    }
}
```

**Note**: This custom policy definition is provided as an example within the Enterprise Policy as Code (EPAC) framework.

### The Key Insight: Resource Group-Level Exclusions

The custom policy handles the AKS scenario through its excludedRG parameter:

```json
"excludedRG": {
    "defaultValue": [
        "MC_*",
        "synapseworkspace-managedrg-*",
        "managed-rg-*", 
        "databricks-*",
        "DefaultResourceGroup*",
        "NetworkWatcherRG",
        "LogAnalyticsDefault*",
        "cloud-shell-storage*"
    ]
}
```
**When AKS tries to create resources:**

- The MC_myakscluster_eastus_12345 resource group matches the MC_* pattern and is excluded.
- All resources created within that resource group are automatically excluded - managed disks, network security groups, public IPs, load balancers, and VMs.
- AKS can manage its entire infrastructure footprint without policy interference.

This resource group-level exclusion is far more efficient than trying to exclude individual resource types. One wildcard pattern handles the entire managed service footprint.

### Policy Evaluation: Runtime vs Assignment Time
This comparison illustrates a fundamental principle: Azure Policy rules are evaluated at resource creation time, not assignment time. The custom policy can make dynamic decisions based on resource properties that don't exist when the policy is assigned.

This is why:

- Built-in policies work well for simple, universal requirements
- Custom policies are needed for complex scenarios with managed services
- Portal-based exemptions can't handle "future" resources that don't exist yet

### Enterprise Policy as Code (EPAC) Framework

The custom policy shown here comes from Microsoft's Enterprise Policy as Code (EPAC) framework, an open-source, Microsoft-curated solution that has been battle-tested in large enterprise organizations. EPAC provides a comprehensive library of custom policy definitions that handle real-world scenarios like managed service exclusions.

The EPAC framework offers significant advantages when tackling this level of policy complexity:

- Pre-built policies that handle common managed service patterns
- Active community support from enterprise practitioners
- Microsoft backing and regular updates
- Proven approaches from large-scale deployments

I can personally attest to the value of the EPAC framework as I have used it to implement and maintain governance in large environment with over 10000 resources, including pretty much every managed service in existence, for close to 3 years.

### Implementation Approaches

**ARM/Bicep Templates:** Deploy custom policies alongside your infrastructure using Infrastructure as Code.

**Terraform/Pulumi:** Include policy definitions as part of your broader infrastructure automation.

**EPAC Framework:** Use the complete enterprise-grade policy management solution.

### When to apply different principles

**Monitor First (Brownfield):**

- Existing environments with established workloads
- Gradual governance implementation
- Learning and discovery phases
- Portal-based management is sufficient

**Custom Policies (Greenfield):**

- New environments requiring immediate enforcement
- Complex exclusion patterns beyond portal capabilities
- Standardized governance across multiple environments
- Enterprise-scale policy management

### The Technical Reality

This progression from portal-based to code-based policy management reflects an important reality: Azure Policy appears simple but quickly becomes complex in real-world scenarios. While basic policies can be managed through the portal, enterprise governance typically requires Infrastructure as Code approaches to handle the full complexity of modern Azure environments.

For organizations implementing sophisticated governance at scale, frameworks like EPAC provide both the technical foundation and the community expertise needed to navigate Azure Policy's complexity successfully.

## Conclusion
Azure Policy appears straightforward but reveals significant complexity when applied to managed services like AKS, Synapse, and Databricks. These services autonomously provision Azure resources in ways that can conflict with governance policies, creating operational challenges that are difficult to predict and troubleshoot.

This article has demonstrated two essential approaches for different scenarios:

For brownfield environments with existing workloads, the "Monitor First" methodology provides a safe path to policy enforcement. By starting with audit mode, evaluating impact, and planning exemptions before enforcement, you can implement governance while avoiding unexpected disruptions.

For greenfield environments or complex exclusion requirements, custom policy definitions with built-in exclusion logic become necessary. The portal's limitations--no wildcard support, static exemptions only--mean that enterprise governance typically requires Infrastructure as Code approaches and frameworks like EPAC.

The progression from simple portal-based policies to sophisticated policy-as-code solutions isn't a failure of the simpler approaches--it's the natural evolution when dealing with the real complexity of modern Azure environments. Understanding when you've outgrown portal-based management is crucial for maintaining both strong governance and operational reliability.

Remember: Good governance enhances operations -- it shouldn't break them. Start with monitoring, progress to custom solutions when needed, and don't hesitate to embrace enterprise frameworks when the complexity demands it.