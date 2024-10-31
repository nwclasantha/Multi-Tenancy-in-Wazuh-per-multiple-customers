## Multi-Tenancy in Wazuh: Enabling Customer-Specific Access to Dashboards and Resources

### Introduction

![image](https://github.com/user-attachments/assets/6d6a324b-96f5-40b4-a117-e50ceafd1248)

Organizations often use Wazuh as a centralized security monitoring platform to manage multiple clients or departments. However, when multiple customers share the same platform, strict data and dashboard segregation is essential to maintain privacy and security. This guide provides a step-by-step approach to configuring Wazuh, Kibana, and Elasticsearch for multi-tenant access. By creating customer-specific roles, dashboards, and implementing Single Sign-On (SSO), each customer can securely view only their relevant resources. This setup ensures isolated access, allowing a scalable and secure shared platform.

---

### Objectives
The objectives of this multi-tenant setup include:
1. **Customer-Specific Access**: Ensure each customer can log in and view only their data and dashboards.
2. **Data Isolation**: Use index patterns to segregate data per customer, preventing unauthorized access.
3. **SSO Integration**: Implement SAML-based SSO to streamline access and simplify user management.
4. **Enhanced Security Compliance**: Maintain strict role-based controls and monitoring to align with privacy and security best practices.

---

### Security Requirements
1. **Role-Based Access Control (RBAC)**: Assign each customer-specific roles in Kibana to ensure only relevant data and dashboards are accessible.
2. **Data Segmentation in Elasticsearch**: Separate customer data at the index level, isolating data to prevent visibility across customers.
3. **SSO for Secure Access**: Use SAML-based SSO to authenticate customers through their Identity Provider (IdP), adding an extra layer of security with support for MFA if available.
4. **Audit Logging**: Monitor and audit access logs in Elasticsearch and Kibana to track unauthorized access attempts and ensure compliance.

---

The detailed, step-by-step guide to setting up Wazuh with WSO2 Identity Server (WSO2 IS) as the SAML-based SSO provider. This setup includes role-based access to ensure each customer only views their respective resources and dashboards.

---

### Prerequisites

1. **Wazuh Stack**: Installed and running (Wazuh Manager, Elasticsearch, Kibana).
2. **WSO2 Identity Server**: Installed, configured, and accessible over HTTPS.
3. **Open Distro Security Plugin for Elasticsearch**: Ensures SAML authentication is supported in Elasticsearch.
4. **SSL Certificates**: HTTPS is required for secure communication in WSO2 IS, Kibana, and Elasticsearch.

---

### Step 1: Configure WSO2 Identity Server for SAML SSO

#### 1.1 Access WSO2 Identity Server (WSO2 IS)
1. Open the WSO2 IS management console by going to `https://<WSO2_SERVER>:9443/carbon` in your browser.
2. Log in with the WSO2 administrator credentials.

#### 1.2 Register Wazuh as a Service Provider (SP) in WSO2 IS
1. In the WSO2 IS management console, navigate to **Main > Identity > Service Providers**.
2. Click **Add** to create a new Service Provider.
3. Enter a unique name for the Service Provider, e.g., `Kibana_Wazuh_SP`.
4. Click **Register** to save the initial setup.

#### 1.3 Configure SAML2 Web SSO for the Service Provider
1. Under the newly created Service Provider, go to **Inbound Authentication Configuration** and click **SAML2 Web SSO Configuration > Configure**.
2. Set the following fields:
   - **Issuer**: Set to `https://kibana.example.com`, matching the SP entity ID in Kibana’s configuration.
   - **Assertion Consumer URL (ACS)**: Enter `https://kibana.example.com/api/security/v1/saml`.
   - **Enable Response Signing**: Check this box to ensure the SAML responses are signed for security.
   - **NameID Format**: Set to `urn:oasis:names:tc:SAML:1.1:nameid-format:unspecified`.

#### 1.4 Configure Attribute Mapping
1. Scroll down to **Attribute Profile** and check **Include Attributes in the Response Always**.
2. Add the following attributes to map roles and email:
   - **Roles**: Map this to `http://wso2.org/claims/role` to use WSO2 IS roles for user access.
   - **Email**: Map this to `http://wso2.org/claims/emailaddress` to identify users uniquely in Wazuh and Kibana.

#### 1.5 Save and Export IdP Metadata
1. Click **Update** to save the SAML configuration for this Service Provider.
2. Go to **View Metadata** and download the IdP metadata XML file. Save this file to use in Elasticsearch configuration.

---

### Step 2: Configure Wazuh’s Elasticsearch for SAML with WSO2 IS

#### 2.1 Place the IdP Metadata File on the Elasticsearch Server
1. Copy the WSO2 IdP metadata XML file to a location accessible by Elasticsearch, such as `/etc/elasticsearch/wso2_idp_metadata.xml`.

#### 2.2 Edit Elasticsearch Configuration (`elasticsearch.yml`)
1. Open `elasticsearch.yml` in the Elasticsearch configuration directory (`/etc/elasticsearch`).
2. Add the following settings to configure WSO2 IS as the SAML IdP:

   ```yaml
   opendistro_security.authc.realms.saml1:
     type: saml
     order: 1
     idp.metadata_file: "/etc/elasticsearch/wso2_idp_metadata.xml"  # Path to the WSO2 IS metadata file
     idp.entity_id: "https://<WSO2_SERVER>:9443/samlsso"  # Replace with your WSO2 IS server's SAML endpoint
     sp.entity_id: "https://kibana.example.com"  # Must match the SP entity ID in WSO2 configuration
     sp.acs: "https://kibana.example.com/api/security/v1/saml"  # Kibana's ACS URL
     kibana_url: "https://kibana.example.com"  # Base URL of Kibana
     roles_key: "http://wso2.org/claims/role"  # Role attribute key from WSO2 IS
     subject_key: "http://wso2.org/claims/emailaddress"  # Email attribute key for user identification
     idp.use_signing: true
   ```

   Ensure all paths, URLs, and attribute keys match the settings configured in WSO2 IS.

#### 2.3 Restart Elasticsearch
Restart the Elasticsearch service to apply the configuration changes:

```bash
sudo systemctl restart elasticsearch
```

---

### Step 3: Configure Kibana for SAML Authentication

#### 3.1 Edit `kibana.yml` for SAML Authentication
1. Open `kibana.yml`, typically located in `/etc/kibana`.
2. Add the following configuration for SAML support:

   ```yaml
   server.xsrf.whitelist: ["/api/security/v1/saml"]
   opendistro_security.auth.type: "saml"
   opendistro_security.cookie.secure: true
   ```

3. **Set Public Base URL**:
   Ensure `server.publicBaseUrl` is set to `https://kibana.example.com`, which should match the `sp.entity_id` in WSO2 IS.

#### 3.2 Restart Kibana
Restart the Kibana service to load the new SAML settings:

```bash
sudo systemctl restart kibana
```

---

### Step 4: Configure Role-Based Access Control (RBAC) in Kibana

#### 4.1 Create Customer-Specific Roles in Kibana

1. **Log in to Kibana as an Administrator**:
   - Navigate to **Management > Security > Roles** in Kibana.

2. **Define Roles for Each Customer**:
   - For each customer, create a role (e.g., `customer_A_role`) and set the following:
     - **Indices**: Specify the customer-specific index pattern, like `wazuh-customer_A-*`.
     - **Privileges**: Grant **read** permissions to view data.
     - **Kibana**: Restrict access to the specific customer dashboard (e.g., `dashboard_customer_A`).

3. **Automate Role Creation via API (Optional)**:
   If managing multiple customers, use the following API to create roles programmatically:

   ```json
   PUT /_security/role/customer_A_role
   {
     "indices": [
       {
         "names": ["wazuh-customer_A-*"],
         "privileges": ["read"]
       }
     ],
     "applications": [
       {
         "application": "kibana-.kibana",
         "privileges": ["read"],
         "resources": ["dashboard_customer_A"]
       }
     ]
   }
   ```

#### 4.2 Map WSO2 IS Roles to Kibana Roles
Use Elasticsearch API to map WSO2 IS roles to the appropriate Kibana roles.

1. **Define Role Mapping via API**:
   - In Kibana Dev Tools or directly in Elasticsearch, create role mappings:

   ```json
   PUT /_security/role_mapping/customer_A_role_mapping
   {
     "roles": [ "customer_A_role" ],
     "rules": {
       "field": { "roles": "customer_A_role" }
     },
     "enabled": true
   }
   ```

---

### Step 5: Create Customer-Specific Dashboards in Kibana

#### 5.1 Create a Base Dashboard Template
1. In **Kibana > Dashboard > Create Dashboard**, create a base dashboard template with common visualizations.
2. Save this dashboard as `Base Dashboard`.

#### 5.2 Duplicate and Customize Dashboards for Each Customer
1. Open `Base Dashboard` and save it as a new dashboard for each customer, e.g., `Customer A Dashboard`.
2. Add a **filter** to restrict data visibility to each customer’s index pattern (e.g., `wazuh-customer_A-*`).
3. Save the customized dashboard.

#### 5.3 Assign Dashboards to Roles
1. Go to **Management > Security > Roles**.
2. For each customer’s role (e.g., `customer_A_role`), assign their specific dashboard (e.g., `Customer A Dashboard`).

---

### Step 6: Test SAML SSO and Customer-Specific Access

#### 6.1 Log into Kibana via WSO2 SSO
1. Open Kibana (`https://kibana.example.com`). This should redirect you to the WSO2 IS login page.
2. Log in with a WSO2 IS user who has the appropriate role assigned.

#### 6.2 Verify Access Control
1. After logging in, confirm that each user can only access their assigned dashboard and data.
2. Verify that no cross-customer access is possible and data isolation is in effect.

#### 6.3 Check Elasticsearch and Kibana Logs
1. Review the Elasticsearch and Kibana logs to ensure there are no authentication or access errors.

---

### Conclusion
This multi-tenancy setup for Wazuh provides customer-specific access and data isolation by combining Kibana RBAC, index pattern segmentation, and SAML-based SSO. Each customer is granted access only to their data and dashboards, enhancing security and data privacy. The inclusion of SSO simplifies user management, allowing customers to log in with their IdP credentials while roles ensure they access only authorized resources. This setup is a scalable, secure solution for organizations aiming to offer shared security monitoring capabilities without compromising individual customer privacy and security compliance.
