## Multi-Tenancy in Wazuh: Enabling Customer-Specific Access to Dashboards and Resources

### Introduction
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

### Configuration Steps

#### Step 1: Set up Customer-Specific Roles in Kibana
To limit each customer’s access, configure roles in Kibana that restrict data visibility to specific dashboards and index patterns.

1. **Access Role Management**:
   - Log in to Kibana as an administrator.
   - Navigate to **Management > Security > Roles**.

2. **Create a Role for Each Customer**:
   - Click **Create Role** and name it (e.g., `customer_A_role`).
   - Under **Indices**, enter the specific index pattern for that customer (e.g., `wazuh-customer_A-*`) and set **Privileges** to **read**.
   - In **Kibana** permissions, restrict access to the customer’s specific dashboard (e.g., `dashboard_customer_A`).
   - Save the role.

3. **Automate Role Creation (Optional)**:
   For multiple customers, use the following API request to streamline role creation:
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

#### Step 2: Create Customer-Specific User Accounts in Kibana
Each customer should have a unique user account assigned to their corresponding role.

1. **Navigate to User Management**:
   - In Kibana, go to **Management > Security > Users**.

2. **Create User Accounts**:
   - Click **Create User**.
   - Enter a username, password, and assign the customer-specific role created above (e.g., `customer_A_role`).
   - Save the user.
   
3. **Example API for User Creation**:
   ```json
   POST /_security/user/customer_A_user
   {
     "password" : "customerA_password",
     "roles" : [ "customer_A_role" ],
     "full_name" : "Customer A User",
     "email" : "customerA@example.com"
   }
   ```

#### Step 3: Create Customer-Specific Dashboards in Kibana
To ensure customers only see their own data, create dashboards with filters applied to their specific index patterns.

1. **Create and Duplicate Dashboards**:
   - In Kibana, go to **Dashboard** and create a base dashboard template.
   - Save this dashboard as `Base Dashboard`.
   - Duplicate it for each customer by clicking **Save As** and naming it according to the customer (e.g., `Customer A Dashboard`).

2. **Apply Customer-Specific Filters**:
   - Open each customer’s dashboard and add a filter to restrict data to the respective customer’s index pattern (e.g., `wazuh-customer_A-*`).
   - Save the dashboard.

3. **Assign Dashboards to Roles**:
   - In **Management > Security > Roles**, assign each customer’s role to their specific dashboard.

#### Step 4: Configure Elasticsearch for Data Segmentation
Using customer-specific index patterns in Elasticsearch allows data isolation at the index level.

1. **Define Index Patterns in Kibana**:
   - Go to **Management > Index Patterns**.
   - Create index patterns for each customer, such as `wazuh-customer_A-*`.

2. **Automate Index Creation (Optional)**:
   To handle multiple customers, use a shell script:
   ```bash
   customers=("customer_A" "customer_B")
   for customer in "${customers[@]}"; do
     curl -X POST "http://localhost:5601/api/saved_objects/index-pattern/${customer}_pattern" \
          -H "kbn-xsrf: true" \
          -H "Content-Type: application/json" \
          -d "{\"attributes\": {\"title\": \"wazuh-${customer}-*\", \"timeFieldName\": \"@timestamp\"}}"
   done
   ```

#### Step 5: Configure Wazuh Agents for Data Tagging
Tags added to each agent’s configuration help filter and route data to the appropriate indices.

1. **Edit the Wazuh Agent Configuration**:
   - Open `ossec.conf` on each customer’s agent.
   - Add a tag for data segregation:
   ```xml
   <agent_config>
     <tag>customer_A</tag>
   </agent_config>
   ```

2. **Logstash Configuration for Data Routing**:
   If using Logstash, set up filters based on tags:
   ```ruby
   filter {
     if "customer_A" in [tags] {
       mutate { add_field => { "[@metadata][target_index]" => "wazuh-customer_A-%{+YYYY.MM.dd}" } }
     }
   }

   output {
     elasticsearch {
       hosts => ["http://localhost:9200"]
       index => "%{[@metadata][target_index]}"
     }
   }
   ```

#### Step 6: Implement SAML-based SSO for Secure Login
Integrating SAML-based SSO simplifies user access and management across multiple customers.

1. **Configure `elasticsearch.yml` for SAML**:
   ```yaml
   opendistro_security.authc.realms.saml1:
     type: saml
     order: 1
     idp.metadata_file: "https://idp.example.com/metadata"
     idp.entity_id: "https://idp.example.com/"
     sp.entity_id: "https://kibana.example.com"
     kibana_url: "https://kibana.example.com"
     sp.acs: "https://kibana.example.com/api/security/v1/saml"
     roles_key: "roles"
   ```

2. **Configure SAML in `kibana.yml`**:
   ```yaml
   server.xsrf.whitelist: ["/api/security/v1/saml"]
   opendistro_security.auth.type: "saml"
   opendistro_security.cookie.secure: true
   ```

3. **IdP Configuration**:
   - Set up SAML in your IdP (e.g., Okta, Azure AD), specifying Kibana’s URL (`https://kibana.example.com`) and ACS endpoint (`https://kibana.example.com/api/security/v1/saml`).

---

### Conclusion
This multi-tenancy setup for Wazuh provides customer-specific access and data isolation by combining Kibana RBAC, index pattern segmentation, and SAML-based SSO. Each customer is granted access only to their data and dashboards, enhancing security and data privacy. The inclusion of SSO simplifies user management, allowing customers to log in with their IdP credentials while roles ensure they access only authorized resources. This setup is a scalable, secure solution for organizations aiming to offer shared security monitoring capabilities without compromising individual customer privacy and security compliance.
