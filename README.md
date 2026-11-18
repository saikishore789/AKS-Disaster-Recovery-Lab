# Azure Kubernetes Service (AKS) – Disaster Recovery Guide

## 1. Introduction

Disaster Recovery (DR) is the process of recovering an application, its data, and its infrastructure when the primary environment becomes unavailable due to a failure or disaster.

For Azure Kubernetes Service (AKS), disaster recovery involves protecting Kubernetes resources, backing up persistent application data, preparing a secondary AKS cluster, and restoring workloads in another Azure region.

The primary objective is to minimize:

- **Data loss:** How much application data can be lost.
- **Downtime:** How long the application remains unavailable.

Azure Backup provides backup and restore capabilities for AKS workloads and supported persistent volumes.

## 2. Scope

This document covers the concepts, prerequisites, architecture, configuration, and recovery procedures required to implement cross-region disaster recovery for Azure Kubernetes Service.

### 2.1 In Scope

The following areas are covered:

1. **AKS Disaster Recovery Architecture**
   - Primary and secondary Azure regions.
   - Primary and DR AKS clusters.
   - Cross-region recovery architecture.

2. **Azure Backup**
   - Backup vault configuration.
   - Backup policies and retention.
   - Operational Tier and Vault-standard Tier.
   - Tiering and geo-redundant storage.
   - Cross Region Restore.

3. **Kubernetes Resource Recovery**
   - Deployments and StatefulSets.
   - Services and Ingress resources.
   - Namespaces, ConfigMaps and Secrets.
   - Persistent Volumes and Persistent Volume Claims.
   - Supported cluster-scoped resources.

4. **Security and Access Management**
   - Trusted Access.
   - Managed identities.
   - Azure RBAC permissions.
   - Backup extension permissions.
   - Storage account access.

5. **Storage and Data Recovery**
   - Operational backup snapshots.
   - Vault Tier storage.
   - Snapshot resource groups.
   - Restore staging storage accounts.
   - Persistent volume recovery.

6. **DR Cluster Preparation**
   - AKS cluster provisioning.
   - Node pools and networking.
   - StorageClass configuration.
   - Required Kubernetes controllers and CRDs.
   - Application dependencies.

7. **Disaster Recovery Execution**
   - Recovery point selection.
   - Cross-region restore.
   - Application validation.
   - DNS and traffic failover.

8. **Recovery Objectives**
   - Recovery Point Objective (RPO).
   - Recovery Time Objective (RTO).
   - DR testing and validation.

### 2.2 Out of Scope

The following activities are not automatically performed by Azure Backup for AKS and are outside the direct workload restore process:

- Automatic recreation of the complete Azure infrastructure.
- Automatic recreation of the target AKS cluster and node pools.
- Automatic recreation of VNets, subnets, NSGs and route tables.
- Automatic recreation of Azure Container Registry.
- Automatic recreation of external databases and services.
- Automatic recreation of Azure Key Vault resources and access policies.
- Automatic DNS and application traffic failover.
- Automatic installation of Istio, Kyverno and other Kubernetes controllers.
- Application-level database replication and continuous transaction log shipping.
- Automatic failback to the original primary region.

These activities must be implemented separately using Infrastructure as Code, application deployment automation, database recovery procedures, and operational processes.

## 3. Assumptions

The disaster recovery architecture and procedures described in this document are based on the following assumptions.

### 3.1 Azure Infrastructure Assumptions

- An active Azure subscription is available.
- The required Azure regions support the selected AKS and Azure Backup capabilities.
- The primary and secondary regions are in the same Microsoft Entra tenant unless a supported alternative configuration is explicitly implemented.
- The secondary region has sufficient compute, networking, storage and service quota.
- The target AKS cluster is provisioned before the restore operation.
- Required Azure infrastructure can be recreated using Terraform, Bicep or other Infrastructure as Code tools.

### 3.2 Azure Backup Assumptions

- An Azure Backup vault is configured for the source AKS cluster.
- The vault uses Geo-Redundant Storage (GRS).
- Cross Region Restore is enabled.
- The backup policy includes Operational Tier and Vault-standard retention.
- An eligible Vault-standard recovery point is available in the secondary region.
- Backup extension and Trusted Access configurations are healthy.
- Required Azure RBAC permissions are assigned to the relevant managed identities.

### 3.3 Kubernetes Assumptions

- The target AKS cluster runs a compatible Kubernetes version.
- Required Kubernetes APIs and Custom Resource Definitions are available.
- Required controllers and operators are installed before restoring their custom resources.
- Application images are accessible from the secondary region.
- Required StorageClasses and CSI drivers are configured.
- Sufficient cluster capacity is available to run restored workloads.
- Application configurations and dependencies are compatible with the target environment.

### 3.4 Storage and Database Assumptions

- Persistent volumes use storage types supported by the selected Azure Backup tier.
- Vault-standard Tier recovery is limited to eligible Azure Disk-based persistent volumes.
- Azure Files volumes are not assumed to be recoverable through Vault-standard Tier.
- The target region has the required disk quota and storage capacity.
- Database recovery procedures account for application consistency.
- Database-native backup or replication is maintained where stronger data consistency or lower RPO is required.

### 3.5 Security Assumptions

- Managed identities are used for service-to-service authentication.
- Least-privilege RBAC permissions are followed.
- Trusted Access is configured between the source Backup vault and target AKS cluster.
- Secrets, certificates and other sensitive configurations are protected.
- Required permissions are available to the administrator performing the recovery.
- Network connectivity to the required Azure services and container registries is available.

### 3.6 Operational Assumptions

- Backup jobs are monitored regularly.
- Recovery points are validated before a disaster occurs.
- DR procedures are documented and maintained.
- Application owners participate in recovery validation.
- RPO and RTO targets are defined by business requirements.
- Disaster recovery exercises are performed periodically.

**Note:** These assumptions describe the expected operating conditions. Actual service support, regional availability, backup eligibility and permissions must be verified against current Microsoft documentation before implementation.

## 4. Disaster Recovery Architecture

A typical cross-region AKS disaster recovery architecture consists of two environments.

### Primary Region

The primary region hosts the production AKS cluster and application workloads.

It includes:

- AKS cluster and node pools.
- Kubernetes Deployments and StatefulSets.
- Services and Ingress resources.
- Persistent Volumes and Persistent Volume Claims.
- ConfigMaps and Secrets.
- Application configuration and security policies.
- Azure Backup extension.
- Azure Backup vault.

### Secondary Region (DR Region)

The secondary region contains the infrastructure required to recover the application.

It includes:

- A pre-provisioned AKS cluster.
- Compatible Kubernetes version.
- Required node pools and networking.
- Azure Backup extension.
- Trusted Access configuration.
- Required managed identities and permissions.
- Compatible StorageClasses.
- Staging storage account.
- Snapshot resource group.
- Application dependencies and container images.

The secondary cluster can receive the restored Kubernetes resources and supported persistent volume data from the primary region's backup recovery point.

## 5. What Is Azure Backup?

Azure Backup is an Azure service that protects workloads by creating recovery points that can be used to restore data and resources after accidental deletion, corruption, or infrastructure failure.

For AKS, Azure Backup protects two major categories:

### 5.1 Kubernetes Resources

Examples include:

- Deployments.
- StatefulSets.
- Services.
- Namespaces.
- ConfigMaps.
- Secrets.
- Ingress resources.
- Persistent Volume Claims.
- Selected cluster-scoped resources.

### 5.2 Persistent Application Data

Azure Backup supports the protection of eligible persistent volumes using supported CSI storage drivers.

For Azure Disk-based persistent volumes, backup protection uses disk snapshots.

The backup can include both Kubernetes resource definitions and the associated persistent volume data.

### Important limitation

Azure Backup does not recreate an entire AKS infrastructure environment.

The target AKS cluster, networking, node pools, identities, ingress controllers, and other external dependencies must be provisioned or configured separately.

## 6. Azure Backup Storage Tiers

Azure Backup for AKS provides two important storage tiers:

1. Operational Tier.
2. Vault-standard Tier.

Each tier has a different purpose.

### 6.1 Operational Tier

Operational Tier is designed for short-term recovery and faster restoration within the same region.

When an AKS backup is created:

1. Kubernetes resource information is captured.
2. Supported persistent volume snapshots are created.
3. Backup metadata is stored in the configured Blob storage.
4. Disk snapshots are stored in the configured snapshot resource group.

Characteristics:

- Supports frequent backups.
- Uses incremental disk snapshots.
- Provides faster operational recovery.
- Stores snapshots in the Azure subscription.
- Supports same-region recovery.
- Does not support cross-region restore.

Typical use cases:

- Accidental deletion of a Deployment.
- Accidental deletion of a StatefulSet.
- Persistent volume recovery.
- Recovery from application configuration changes.

### 6.2 Vault-standard Tier

Vault-standard Tier is designed for long-term retention and disaster recovery.

Eligible Operational Tier backups are transferred into the Backup vault's managed storage.

Characteristics:

- Provides long-term backup retention.
- Stores backup data outside the customer's Azure tenant.
- Supports geo-redundant protection when configured.
- Supports cross-region restore when the required redundancy and recovery settings are enabled.
- Supports Azure Disk-based persistent volumes subject to service limitations.
- Does not support Azure Files volumes in Vault Tier.

Typical use cases:

- Regional Azure outages.
- Long-term application recovery.
- Protection against deletion of operational backup resources.
- Disaster recovery into another Azure region.

### 6.3 Operational Tier vs Vault-standard Tier

| Feature | Operational Tier | Vault-standard Tier |
|---|---|---|
| Purpose | Short-term recovery | Long-term protection |
| Storage | Operational snapshots and backup metadata | Vault-managed backup storage |
| Backup frequency | 4, 6, 8, 12 or 24 hours | One eligible scheduled recovery point per day |
| Incremental disk snapshots | Yes | Derived from operational backups |
| Same-region restore | Yes | Yes |
| Cross-region restore | No | Yes, with required configuration |
| Long-term retention | Limited | Supported |
| Protection outside tenant | No | Yes |

## 7. What Is Tiering?

Tiering is the process of transferring an eligible Operational Tier backup into the Vault-standard datastore.

It allows a backup recovery point to receive long-term protection and become eligible for cross-region recovery.

### Tiering workflow

1. An AKS backup is created.
2. Kubernetes resource information is captured.
3. Persistent volume snapshots are created.
4. The backup is stored in Operational Tier.
5. Azure evaluates the backup against the configured Vault-standard retention rule.
6. Eligible recovery points enter the tiering process.
7. Backup data is transferred to the Vault-standard datastore.
8. The recovery point is replicated to the secondary region when geo-redundant protection is configured.

### Tiering delay

Azure documentation states that tiering can take up to four hours to begin after a backup becomes eligible.

This does not mean that the entire transfer completes within four hours.

The actual completion time depends on the backup size and processing.

### Vault Tier retention rule

A commonly used retention rule is:

**First successful backup taken every day.**

This rule makes the first successful scheduled backup of the day eligible for Vault-standard retention.

Example configuration:

| Setting | Example |
|---|---|
| Operational backup frequency | Every 4 hours |
| Operational retention | 7 days |
| Vault-standard retention | 30 days |
| Vault retention rule | First successful backup taken every day |

Changing the backup policy does not automatically convert existing Operational Tier recovery points into Vault-standard recovery points.

A new eligible backup is required.

## 8. Geo-Redundant Storage (GRS)

Geo-Redundant Storage is an Azure storage redundancy option that maintains copies of data in a secondary paired Azure region.

It provides additional protection against regional failures.

For AKS cross-region disaster recovery, the Backup vault must be configured with the appropriate geo-redundant storage option.

### Cross Region Restore (CRR)

Cross Region Restore allows supported vaulted backup recovery points to be restored from the secondary region.

It is important to understand:

- GRS provides geo-redundant data protection.
- CRR enables recovery using the secondary-region copy.
- Operational Tier recovery points cannot be used for cross-region restore.
- Vault-standard recovery points are required for this recovery workflow.

GRS and CRR must be configured appropriately before a regional disaster occurs.

## 9. Prerequisites for Cross-Region AKS Recovery

Several infrastructure, security, and storage prerequisites must be satisfied.

### 9.1 Backup Vault Prerequisites

- Backup vault must be configured in the source region.
- Backup vault must use geo-redundant storage.
- Cross Region Restore must be enabled.
- AKS backup policy must include Vault-standard retention.
- An eligible Vault-standard recovery point must exist.
- The recovery point must be available in the secondary region.

### 9.2 Source AKS Prerequisites

- AKS cluster must be supported by Azure Backup.
- Azure Backup extension must be installed.
- Backup extension must be healthy.
- Trusted Access must be enabled.
- Persistent volumes must use supported storage drivers.
- Backup policy must include the required namespaces and resources.

### 9.3 Target AKS Prerequisites

- A target AKS cluster must already exist.
- The cluster must be in the secondary region.
- Kubernetes version must be compatible.
- Node pools must have sufficient capacity.
- Required managed identities must exist.
- Azure Disk CSI driver must be available.
- Compatible StorageClasses must exist.
- Backup extension must be installed.
- Trusted Access must be configured.
- Required networking and storage resources must be available.

### 9.4 Storage Prerequisites

The restore process uses different storage resources for different purposes.

**Backup extension storage account**

Stores backup metadata and Kubernetes resource backup data in a Blob container.

**Snapshot resource group**

Contains disk snapshots used for operational backup and restore.

**Restore staging storage account**

Temporarily stores hydrated backup data during Vault Tier restoration.

**Restore staging resource group**

Contains resources required during the Vault Tier restore workflow.

For cross-region recovery, staging resources and snapshot resources must be configured in the target region according to the restore workflow requirements.

### 9.5 Application Prerequisites

Before restoring application workloads, prepare:

- Required Kubernetes CRDs.
- Ingress controllers.
- Service mesh components.
- Security policy engines.
- StorageClasses.
- Container images.
- Secrets and certificates.
- External dependencies.
- Monitoring and logging components.

Restoring Kubernetes custom resources does not automatically install their controllers or CRDs.

## 10. Trusted Access

Trusted Access is an Azure AKS feature that allows supported Azure services to communicate with an AKS cluster through an Azure-managed access mechanism.

For Azure Backup, Trusted Access allows the source Backup vault to access the target AKS cluster for restore operations.

It avoids relying on manually managed Kubernetes administrator credentials for this integration.

### Trusted Access components

**Source resource**

The Azure Backup vault.

**Target resource**

The destination AKS cluster.

**Role**

The predefined backup operator role:

`Microsoft.DataProtection/backupVaults/backup-operator`

### Trusted Access configuration

Trusted Access is configured by creating a role binding on the target AKS cluster.

The binding references:

- Target AKS cluster.
- Source Backup vault resource ID.
- Backup operator role.

### Important distinction

Trusted Access and Azure RBAC are different.

Trusted Access provides the service-to-cluster access relationship.

Azure RBAC controls permissions on Azure resources such as storage accounts, resource groups, and disk snapshots.

Both are required for a successful restore.

## 11. Managed Identities and Azure RBAC

Azure Backup uses managed identities to access Azure resources securely without storing usernames or passwords.

Different identities perform different operations.

### 11.1 Backup Vault Managed Identity

The Backup vault normally uses a system-assigned managed identity.

It requires the following permissions for Vault Tier restoration:

| Resource scope | Required role |
|---|---|
| Target AKS cluster | Reader |
| Staging resource group | Contributor |
| Staging storage account | Storage Account Contributor |
| Staging storage account | Storage Blob Data Owner |

For Operational Tier restoration, additional or different permissions apply to the target cluster and snapshot resource group.

### 11.2 Target AKS Managed Identity

The target AKS cluster can use a system-assigned or user-assigned managed identity.

For the documented restore workflow, the AKS identity requires:

| Resource scope | Required role |
|---|---|
| Target snapshot resource group | Contributor |

This allows the target cluster identity to manage the disk snapshot resources needed during restore.

### 11.3 Backup Extension Managed Identity

The Backup extension has its own managed identity.

For Vault Tier restoration:

| Resource scope | Required role |
|---|---|
| Staging storage account | Storage Blob Data Contributor |

For Operational Tier restoration, the extension identity requires appropriate Blob permissions on the operational backup storage account.

### 11.4 Permission Summary

| Identity | Scope | Role |
|---|---|---|
| Backup vault identity | Target AKS cluster | Reader |
| Backup vault identity | Staging resource group | Contributor |
| Backup vault identity | Staging storage account | Storage Account Contributor |
| Backup vault identity | Staging storage account | Storage Blob Data Owner |
| Target AKS identity | Snapshot resource group | Contributor |
| Backup extension identity | Staging storage account | Storage Blob Data Contributor |

The exact required assignments depend on the selected restore tier and workflow.

Use the Azure Backup portal's View Required Roles and Grant Permissions features to verify the current requirements.

### 11.5 RBAC Propagation

Azure role assignments may take time to propagate.

After creating role assignments:

1. Wait for propagation.
2. Verify the assignments.
3. Revalidate the restore operation.
4. Review the Azure Activity Log if validation still fails.

Do not repeatedly create duplicate role assignments.

## 12. Resources Recovered by Azure Backup

Azure Backup can restore selected Kubernetes resources and supported persistent volume data.

### Kubernetes resources

Examples:

- Namespaces.
- Deployments.
- StatefulSets.
- Services.
- ConfigMaps.
- Secrets.
- Persistent Volume Claims.
- Ingress resources.
- Selected cluster-scoped resources.

### Persistent data

Supported persistent volume data can be restored using the appropriate backup datastore and supported storage driver.

### Resources that must be recreated separately

Azure Backup does not automatically recreate:

- AKS control plane.
- Node pools and virtual machine scale sets.
- Virtual networks and subnets.
- Network security groups.
- Route tables.
- Azure Container Registry.
- External load balancers and public IP configuration.
- Azure Key Vault.
- Managed identities.
- DNS records.
- Ingress controller installations.
- Service mesh installations.
- Security policy engines.
- Monitoring infrastructure.
- External databases.

These should be managed through Infrastructure as Code and configuration management.

## 13. Cross-Region Restore Procedure

The general recovery workflow consists of the following phases.

### Phase 1: Prepare the Primary Environment

1. Create and configure the AKS cluster.
2. Deploy application workloads.
3. Configure persistent storage.
4. Install the Backup extension.
5. Configure Trusted Access.
6. Create the Azure Backup policy.

### Phase 2: Configure Backup Protection

1. Configure GRS.
2. Enable Cross Region Restore.
3. Configure Operational Tier retention.
4. Configure Vault-standard retention.
5. Select the required namespaces and resources.
6. Include supported persistent volumes.
7. Run a backup.

### Phase 3: Wait for Vault Tier Availability

1. Verify successful backup completion.
2. Confirm that the backup qualifies for Vault-standard retention.
3. Wait for tiering to begin.
4. Wait for Vault Tier transfer to complete.
5. Wait for secondary-region replication.
6. Verify the recovery point is available in the secondary region.

### Phase 4: Prepare the DR Cluster

1. Provision the target AKS cluster.
2. Configure node pools.
3. Configure networking.
4. Install the Backup extension.
5. Configure Trusted Access.
6. Configure managed identities.
7. Assign the required RBAC roles.
8. Create staging and snapshot resources.
9. Install required CRDs and application dependencies.
10. Configure compatible StorageClasses.

### Phase 5: Initiate Restore

1. Open the Backup vault.
2. Navigate to the protected AKS backup instance.
3. Select the recovery operation.
4. Select Secondary Region.
5. Select an eligible Vault-standard recovery point.
6. Select the target AKS cluster.
7. Select the resources and namespaces.
8. Configure staging storage.
9. Configure the snapshot resource group.
10. Run Backup Readiness Validation.
11. Review and initiate restore.

### Phase 6: Validate the Application

After restore:

1. Verify Kubernetes resources.
2. Verify Deployments and StatefulSets.
3. Verify Service endpoints.
4. Verify PVC and PV status.
5. Verify persistent application data.
6. Verify application connectivity.
7. Verify ingress and load balancing.
8. Verify security policies.
9. Verify monitoring and logging.
10. Update DNS to route traffic to the DR environment.

## 14. RPO – Recovery Point Objective

RPO stands for Recovery Point Objective.

It defines the maximum amount of data loss an organization is willing to tolerate following a disaster.

In simple terms:

**RPO answers: How much data can we afford to lose?**

### Example

Assume:

- Last successful backup: 10:00 AM.
- Disaster occurs: 2:00 PM.
- Recovered backup: 10:00 AM.

The potential data loss window is four hours.

Therefore, the example RPO is four hours.

### RPO and backup frequency

| Backup frequency | Potential recovery point gap |
|---|---|
| Every 4 hours | Approximately 4 hours |
| Every 6 hours | Approximately 6 hours |
| Every 12 hours | Approximately 12 hours |
| Every 24 hours | Approximately 24 hours |

These are scheduling intervals, not guarantees of actual recovery-point availability.

### Vault Tier RPO

For the documented AKS Vault Tier cross-region workflow:

- One successful scheduled backup per day is eligible for Vault-standard retention.
- Secondary-region replication can take up to another 12 hours.
- The documented secondary-region RPO can be up to 36 hours.

This means that Vault Tier alone may not satisfy applications requiring very low data loss.

For databases such as PostgreSQL, consider database-native replication, continuous log archiving, and transaction log recovery.

## 15. RTO – Recovery Time Objective

RTO stands for Recovery Time Objective.

It defines the maximum amount of time an organization can tolerate application downtime after a disaster.

In simple terms:

**RTO answers: How long can the application remain unavailable?**

### Example

Assume:

- Disaster occurs at 10:00 AM.
- DR restoration starts at 10:15 AM.
- Application becomes available at 12:00 PM.

The RTO is two hours from the time of the disaster.

### Activities affecting RTO

| Activity | Example duration |
|---|---|
| Disaster declaration | 10 minutes |
| Infrastructure preparation | 20 minutes |
| Permission validation | 10 minutes |
| Workload restoration | 20 minutes |
| Persistent volume recovery | 25 minutes |
| Application validation | 10 minutes |
| DNS failover | 5 minutes |

These durations are illustrative and must be measured through actual DR exercises.

### How to reduce RTO

- Pre-provision the DR AKS cluster.
- Use Terraform for infrastructure provisioning.
- Use GitOps for Kubernetes configuration.
- Keep container images accessible in the DR region.
- Automate application dependency installation.
- Automate DNS failover.
- Maintain tested database recovery procedures.
- Run regular disaster recovery exercises.

## 16. RPO vs RTO

| Parameter | RPO | RTO |
|---|---|---|
| Full form | Recovery Point Objective | Recovery Time Objective |
| Measures | Data loss | Application downtime |
| Main question | How much data can we lose? | How long can the application remain unavailable? |
| Example | 4 hours | 2 hours |
| Improved by | More frequent backups and replication | Automation and pre-provisioned infrastructure |

RPO and RTO are separate business continuity objectives.

A solution may have a low RTO but a high RPO, or a low RPO but a high RTO.

Both must be designed and tested independently.

## 17. Limitations and Important Considerations

### Azure Backup limitations

- Operational Tier does not support cross-region restore.
- Vault Tier requires an eligible recovery point.
- Vault Tier supports only specific persistent volume types.
- Azure Files volumes are not supported in Vault Tier.
- Disk size and access limitations apply.
- AKS infrastructure is not automatically recreated.
- Required CRDs and controllers must exist before restoring custom resources.
- Backup policy changes do not retroactively convert old recovery points.

### Application considerations

- Database snapshots are not automatically application-consistent.
- Stateful applications require application-aware recovery testing.
- Secrets and certificates must be protected.
- External services must be available in the DR region.
- DNS and application traffic must be switched.
- StorageClass compatibility must be verified.

### Security considerations

- Use managed identities instead of stored credentials.
- Follow least-privilege RBAC.
- Keep backup vaults protected from unauthorized deletion.
- Protect recovery points from accidental deletion.
- Restrict access to sensitive Kubernetes Secrets.
- Audit all restore operations.

## 18. Disaster Recovery Validation Checklist

### Backup configuration

- [ ] Backup vault configured.
- [ ] GRS enabled.
- [ ] Cross Region Restore enabled.
- [ ] Operational Tier retention configured.
- [ ] Vault-standard retention configured.
- [ ] Required namespaces selected.
- [ ] Persistent volumes included.
- [ ] Backup job completed successfully.
- [ ] Vault recovery point available.
- [ ] Secondary-region replication completed.

### DR infrastructure

- [ ] Target AKS cluster provisioned.
- [ ] Node pools healthy.
- [ ] Networking configured.
- [ ] StorageClass created.
- [ ] Backup extension installed.
- [ ] Trusted Access configured.
- [ ] Managed identities verified.
- [ ] Required RBAC roles assigned.
- [ ] Staging storage account configured.
- [ ] Snapshot resource group configured.

### Application recovery

- [ ] Required CRDs installed.
- [ ] Ingress controller installed.
- [ ] Security policies installed.
- [ ] Container images accessible.
- [ ] Kubernetes workloads restored.
- [ ] Services restored.
- [ ] PVCs bound.
- [ ] Persistent data verified.
- [ ] Database connectivity verified.
- [ ] Application endpoints verified.
- [ ] DNS updated.
- [ ] Monitoring and alerting operational.

### DR exercise

- [ ] RPO measured.
- [ ] RTO measured.
- [ ] Restore job successful.
- [ ] Application traffic successfully switched.
- [ ] Recovery documentation updated.
- [ ] DR exercise results documented.

## 19. Recommended Disaster Recovery Best Practices

1. Use Infrastructure as Code to recreate infrastructure.
2. Keep the DR cluster pre-provisioned for critical workloads.
3. Use Azure Backup with GRS and Vault-standard retention.
4. Maintain a separate database-aware backup strategy.
5. Store application configuration in version control.
6. Use GitOps for application deployment.
7. Keep container images accessible from the secondary region.
8. Use managed identities and least-privilege RBAC.
9. Protect backup recovery points against accidental deletion.
10. Automate DNS failover.
11. Monitor backup jobs and replication status.
12. Perform regular restore tests.
13. Measure actual RPO and RTO during DR exercises.
14. Document failover and failback procedures.
15. Test recovery of persistent data, not just Kubernetes resource definitions.

## 20. Conclusion

AKS disaster recovery requires more than creating backups.

A complete solution combines:

- Kubernetes workload protection.
- Persistent volume backup.
- Operational Tier for short-term recovery.
- Vault-standard Tier for long-term and cross-region protection.
- Geo-redundant backup storage.
- Trusted Access.
- Managed identities and Azure RBAC.
- Pre-provisioned DR infrastructure.
- Application dependency recovery.
- DNS and traffic failover.
- RPO and RTO measurement.

The ultimate goal is to ensure that applications and their data can be recovered in another Azure region within the organization's defined recovery objectives.

**Remember:**

- Operational Tier = Short-term operational recovery.
- Vault-standard Tier = Long-term backup and cross-region recovery.
- Tiering = Transfer of eligible backups into Vault-standard storage.
- GRS = Geo-redundant data protection.
- CRR = Recovery from the secondary region.
- RPO = Acceptable data loss.
- RTO = Acceptable application downtime.

## 21. Microsoft Documentation

- [Azure Backup for AKS – Overview](https://learn.microsoft.com/en-us/azure/backup/azure-kubernetes-service-backup-overview)
- [AKS Backup – Support Matrix](https://learn.microsoft.com/en-us/azure/backup/azure-kubernetes-service-cluster-backup-support-matrix)
- [AKS Backup – FAQ](https://learn.microsoft.com/en-us/azure/backup/azure-kubernetes-service-backup-faq)
- [AKS Backup – Configure Backup](https://learn.microsoft.com/en-us/azure/backup/azure-kubernetes-service-cluster-backup-using-cli)
- [Restore AKS Backups Across Regions](https://learn.microsoft.com/en-us/azure/backup/tutorial-restore-aks-backups-across-regions)
- [AKS Backup – Restore](https://learn.microsoft.com/en-us/azure/backup/azure-kubernetes-service-cluster-restore)
- [AKS Trusted Access](https://learn.microsoft.com/en-us/azure/aks/trusted-access-feature)

