# demo
## Step-by-Step Deployment

Deploy one environment at a time. Don’t use `scripts/deploy-all.sh` for the first rollout; review each plan separately, especially production.

1. **Plan the AWS foundation.** Confirm this is a new Organization; bootstrap creates one and should not be run against an existing Organization. Decide account emails, regions, unique VPC CIDRs, account owners, and which accounts may communicate.

2. **Prepare bootstrap values.** From the repository root, copy the bootstrap example and replace every placeholder:

   ```bash
   cp bootstrap/terraform.tfvars.example bootstrap/terraform.tfvars
   ```

   Set the real management account ID, unique member-account emails, and globally unique state bucket name. Confirm your active AWS identity is the management account:

   ```bash
   aws sts get-caller-identity
   ```

3. **Create the Organization and backend.** Run the guarded bootstrap script:

   ```bash
   scripts/bootstrap-state.sh
   ```

   Review the plan. Type `APPLY BOOTSTRAP` only if it matches the intended accounts, Organization, and state resources. The script migrates bootstrap state to S3 after creation.

4. **Configure each environment’s backend.** Get the actual backend values:

   ```bash
   terraform -chdir=bootstrap output -raw state_bucket_name
   terraform -chdir=bootstrap output -raw lock_table_name
   ```

   Put those values in each `environments/*/backend.hcl`, with a **unique `key` per environment**. Include the correct region.

5. **Create the deployment roles.** Create a least-privilege Terraform execution role in every target account. Its trust policy must allow your deployment identity to assume it. The project uses these roles but does not create them. Confirm the role ARN and expected account ID for each environment.

6. **Prepare local environment variables.** Copy each environment’s example, then replace its placeholders. For example:

   ```bash
   cp environments/network/terraform.tfvars.example environments/network/terraform.tfvars
   cp environments/shared-services/terraform.tfvars.example environments/shared-services/terraform.tfvars
   ```

   Repeat for management, security, sandbox, and each workload root you plan to deploy. Set each `expected_account_id` and `terraform_role_arn`. Supply passwords through a secrets-backed process; remove placeholder password lines from local tfvars if injecting secrets with `TF_VAR_...`. Don’t commit local tfvars files.

7. **Deploy management and security.** Enable IAM Identity Center and create the groups referenced by the management example, then deploy:

   ```bash
   scripts/deploy-environment.sh management
   scripts/deploy-environment.sh security
   ```

   Review each plan before typing its approval phrase. Note that centralized CloudTrail and attached SCPs are not implemented yet.

8. **Deploy the network hub.** Set `transit_gateway_spoke_account_ids` to the real spoke account IDs, then deploy:

   ```bash
   scripts/deploy-environment.sh network
   terraform -chdir=environments/network output -raw transit_gateway_id
   ```

   Copy that TGW ID into the local tfvars for shared-services and each workload environment. Set `transit_route_cidrs` to only approved destinations.

9. **Deploy shared services, then non-production.** Set the real DNS zone and directory secret, then deploy in order:

   ```bash
   scripts/deploy-environment.sh shared-services
   scripts/deploy-environment.sh sandbox
   scripts/deploy-environment.sh workload-dev
   scripts/deploy-environment.sh workload-stage
   ```

   Test connectivity and logging. Cross-account private-DNS association is not implemented; the current private zone is attached only to the shared-services VPC.

10. **Hold production until controls are ready.** The current TGW uses automatic association and propagation on its default route table, so it doesn’t provide network segmentation or inspection. Before production, implement and test segmented routing and firewall inspection, central CloudTrail/log archive, and required SCPs. Then deploy:

   ```bash
   scripts/deploy-environment.sh workload-prod
   ```

   Review the production plan carefully before approving it.

The repository-root `terraform plan` is a separate stack for `tagging-policy.tf`; it does not plan the bootstrap or environment roots. Use `terraform validate` for syntax checks, and `scripts/deploy-environment.sh <environment>` when you’re ready for a reviewed plan and explicitly approved apply.



================================================================================
for a new organisation starting to its infrastructure
================================================================================
For a new organization, start with the **governance, identity, state, and audit foundation**. Add workload accounts only when the company is ready to operate them safely. This repository is a scaffold, so several controls described in its docs—especially centralized CloudTrail, SCP attachment, and network inspection—still need implementation before production use.

**Initial foundation**

1. **Decide account and network boundaries.** Assign owners for management, security/log archive, network, shared services, and workloads. Reserve non-overlapping CIDRs and select the regions you’ll allow.
2. **Bootstrap the Organization and Terraform state.** Run the guarded bootstrap only if this is a new AWS Organization. It creates accounts, OUs, and the S3/KMS/DynamoDB state foundation in bootstrap/main.tf.
3. **Establish access.** Create least-privilege Terraform execution roles in each target account, and enable IAM Identity Center with the groups required by environments/management.
4. **Build security and audit first.** Deploy the security environment, then implement organization-wide CloudTrail delivery to a protected log archive and attach reviewed SCPs. The current security root provides some services, but CloudTrail and SCP enforcement aren’t implemented.
5. **Build the network hub.** Deploy the network account’s inspection VPC and Transit Gateway, then connect only approved spokes. The current TGW default route table can permit spoke-to-spoke routing; it is not a firewall or isolation boundary.
6. **Start with non-production.** Deploy shared services if needed, then sandbox and development. Validate identity, DNS, routes, logs, backup, and recovery before introducing production.

**As the company grows**

- Add staging and production accounts when there is a clear deployment and support process.
- Automate account creation and baseline configuration, so every new account receives its roles, logging, security services, tags, budgets, and network settings consistently.
- Add separate policies and network route tables for different trust zones; route sensitive traffic through an inspection firewall.
- Add cross-account private DNS associations and resolver-rule sharing when workloads need shared name resolution.
- Expand regions and services through reviewed changes, with separate Terraform state per account/environment and approval-gated CI plans.

One caveat: the current bootstrap creates all member accounts, including production, staging, dev, and sandbox, at once. If the organization wants a smaller initial footprint, make those account resources optional before running bootstrap. Avoid applying `workload-prod` until the audit, access, backup, and network controls are implemented and tested.