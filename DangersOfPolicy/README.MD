# Monitor First: Protecting your managed services from Azure Policy Pitfalls

*How enforcing Azure Policies without proper planning can silently break managed services in Azure and the proven approach to avoid it*

## Introduction

Azure Policy provides powerful governance capabilities across your Azure environment, but when implemented incorrectly, it can have devastating consequences for your managed services. Without proper planning and testing, restrictive policies can silently break critical functionality in services like Azure Kubernetes Service (AKS), Azure Synapse, and Azure Databricks.

In this article, we'll demonstrate how careless policy deployments can cause unexpected downtime and operational failures. Using AKS as a detailed case study, we'll show exactly what goes wrong when policies with DENY effects are applied without consideration for managed services that automatically provision Azure resources.

We'll walk through a real-world scenario where a simple tagging policy breaks Kubernetes deployments, explain why this happens, and most importantly--show you the "Monitor First" approach that prevents these issues.

After reading this article, you'll understand how to safely implement Azure Policy in environments with managed services, recognize the warning signs of policy-related failures, and have a proven methodology to avoid costly downtime while maintaining strong governance.

### Who needs to understand this?

This article is essential if you:
- Are planning to implement Azure Policy in environments with managed services
- Have experienced unexplained failures after policy deployments
- Manage AKS, Synapse, Databricks, or other services that auto-provision resources
- Need to balance governance requirements with operational reliability
- Want to avoid costly downtime from policy-related service disruptions

### Two Approaches to Policy Implementation:

**The Risky Approach: Enforce First**
- Quick to implement
- Instant enforcement
- BUT: Can cause unexpected service disruptions
- Hard to troubleshoot when things break

**The Safe Approach: Monitor First**
- Requires initial planning
- Delayed enforcement
- BUT: Avoids operational surprises
- Provides clear impact assessment

![Two Approaches](/DangersOfPolicy/img/1-twoapproaches-EN.png)

## The Problem: Azure Policy's Impact on Managed Service Operations

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

To demonstrate this issue, I set up a test focused on AKS, though the lessons apply broadly:

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

## Conclusion

Azure Policy is an essential governance tool, but applying restrictive effects without understanding their impact on managed services can lead to serious operational issues. From AKS to Synapse, Databricks, and other services that autonomously provision Azure resources, policies can silently break functionality in ways that are difficult to troubleshoot.

By adopting a "Monitor First" approach, you can implement strong governance while avoiding unexpected disruptions across your Azure environment. Always begin with audit mode, understand the impact, and plan accordingly before enforcing policies that might affect managed services.

**Remember: Good governance enhances operations--it shouldn't break them. Monitor first, enforce second.**