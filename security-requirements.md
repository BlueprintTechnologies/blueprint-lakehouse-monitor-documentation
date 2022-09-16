## Security Requirements for VM/AppService runtime
Here you will find the security requirements for both the Azure App Service and Azure Virtual Machine deployment. The requirements are separated into four separate phases.

### Phase 1) Azure Resources
After the deployment of the Lakehouse Monitor is complete, the provided resource group will include the following resources:

* **VM or AppService**: includes the reporting and instrumentation dashboard, DBricks workspace configuration panel, background services for telemetry data analysis and recommendations, as well as consumption data scheduled runs

* **Storage Account**: used for storing all telemetry data from the Databricks workspaces and consumption/cost/usage detail data

* **KeyVault**: used for storing the storage account access key, the Azure AD App Registration client secret (for Single Sign On into the app) as well as the token store sas uri secret

* **SQL Server database**: used for storing the output of the analyzer and consumption data processors, feeds all the data required by the reports and dashboards


Signing into the application url is done through Azure AD credentials that are authorized for access into all available subscriptions in the Azure tenant and all the Databricks workspaces in the available subscriptions.

### Phase 2) Sign-in user requirements
The user who installed the Lakehouse Monitor application is assumed to already have `User Access Administrator` role. This role is required to configure the following required permissions for the Managed Identity of the AppService or System Assigned Managed Identity of the VM from the Azure Portal.

Each signed-in Azure AD user must have following permissions in order use (e.g. to attach/detach the telemetry data collector) on Databricks resources(e.g. jobs, clusters):

* `Microsoft.Databricks/workspaces/read` permission via a custom role at either Azure subscription or resource group level containing the Databricks workspaces this user should be able to access from the application
    * provide the above role from Azure Portal by configuring either the Azure subscription or the resource group
    * alternatively, if the role with the custom permission cannot be created and assigned in your org, the subscriptionMetadata.csv file can be created manually and made available as fallback. Contact Blueprint with support on this matter.

* A `Databricks user` must be created in the Workspace/Settings/Admin Console/Users for the signed in Azure AD user. The app does not create them automatically as it would require a higher privilege for the signed in user. 

    * The Databricks user’s permission will dictate the level of access to resources. If the workspace is part of the Premium tier, more granular access control can be enabled, and the “Can Manage” permission is required for this user to change cluster or job configuration. 

    * For non-Premium workspaces, any Databricks user can edit configurations, the one exception is All Purpose clusters, they can be only edited by their Owners, since we introduced Databricks secrets in configuration.

### Phase 3) Access roles configuration 
The signed in user will grant the application (AppService or VM) the necessary permissions to load consumption data on a schedule and analyze telemetry data. The signed in user must have at least the **UserAccessAdministrator** role in the subscription.

The Managed identity of the AppService or System Assigned Managed Identity of the VM are going to be used to perform these background processes, these are the following roles they are being granted by the signed in user: 

* `Billing Reader` role at Azure subscription level in order to allow the application to read consumption/usage details data. This operation can be done by clicking the Grant Billing Reader role button on settings page for selected Azure subscription in the Lakehouse Monitor user interface

* Create a Databricks user for the Managed Identity in the selected Databricks workspace in order to allow the application to access the Databricks job/cluster configuration (REST APIs) for background processors. This operation can be done by clicking `Grant Access` button on settings page for selected Databricks workspace in the Lakehouse Monitor user interface. The created user is part of the admins group in the workspace.

* Databricks requires each Azure managed identity service user to have a correspondent user in Databricks in order to use the APIs for listing and managing jobs or clusters. For a non-premium Databricks workspace, the only way to create automatically a corresponding user is to allow the managed identity service user to create the user in Databricks. This is done by temporary granting the `Contributor` role to the managed identity service user. Once this corresponding service user is created in Databricks, the `Contributor` role is removed and only `Reader` role is maintained for the managed identity.

* In order to use the Service Principal of the application (used for SSO) in Databricks notebooks and scheduled jobs, click on the 'Add Service Principal' button from Settings page. A Databricks user will be created for the SP in the selected workspace, part of the admins group.

### Phase 4) Secret Scope creation

The telemetry collector agent or the service principal (if used) are using Databricks Secrets to for securely accessing the cloud storage configuration, including the access key.

Click the `Create Secret Scope` button on settings page for selected Databricks workspace in the Lakehouse Monitor user interface.

Both Azure KeyVault backed secret scopes and Databricks backed secret scopes are supported:

* Databricks backed secret scope: signed in AD user must be part of the admins group in the workspace

* Azure KeyVault backed secret scope:  signed in Azure AD user should  have both the `Key Vault Contributor` role for the KeyVault used for the selected workspace and an access policy with the Set permission enabled in that KeyVault