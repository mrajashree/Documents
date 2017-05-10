<h1> PingFederate Configuration </h1>

1. After installing PingFederate, select the option to configure PingFederate as an Identity Provider.

2. Go to **Server Configuration**. Click on **System Settings** -> **Data Store**s. In the Manage Data Stores page, click on **Add New Data Store**. 

	a. Data Store Type: In our example, we'll use the **LDAP** type. Select **LDAP** and click on **Next**. 
	
	b. LDAP Configuration: Provide details regarding your LDAP connection. The configuration will require the following infromation: HOSTNAME, LDAP TYPE (Active Directory in this doc), USER DN, PASSWORD. When you click on **Next**, it will attempt to connect to your LDAP server. 

<h2> If the Service Provider connection is not created </h2>
<h3> Generating Identitiy Provider metadata </h3>

1. Go To **Server Configuration**. Click on **Administrative Functions** -> **Metadata Export**.  
	a. Metadata Mode: Pick **Select Information to include in Metadata manually** and click on **Next**.
	
	b. Protocol: The protocol is already pre-selected, so click on **Next**.
	
	c. Attribute Contract: These are the attributes that will need to configured in order to be used when configuring Access Control in Rancher. There are four attributes that need to be added:
	
		1. cn
		2. displayName
		3. givenName
		4. memberOf

Example of what the Attribute Contract page should look like after adding all 4 fields:
![Attribute Contract IdP](https://github.com/mrajashree/Documents/blob/master/images/IdP-metadata-creation.png)

2. Complete the rest of the steps to generate metadata, export it and save as PingIdP_metadata.xml

<h2> Pre-existing SP connection </h2>

1. If SP connection is already created for PingFederate IdP configuration, to get the IdP metadata, go to 
**Server Configuration**. Click on **ADMINISTRATIVE FUNCTIONS** -> **Metadata Export** ->  USE A CONNECTION FOR METADATA GENERATION, and use the existing connection from dropdown presented

2. Make sure the SP Connection's following fields are set correctly:
* Partner's Entity ID (Connection ID)
* Base URL
* Assertion Consumer Service URL (Endpoint)

<h3> Generating Service Provider (Rancher) metadata </h3>

Before proceeding with these steps, make sure the host setting is saved correctly. Go to Admin -> Settings -> Host Registration URL -> Choose 'This site's address' and hit Save. In absense of this, metadata generated will be incorrect

1. On Rancher UI, enter the first four fields under Shibboleth access control based on your attribute contract from previous section. 
![Rancher Access Control configuration attributes](https://github.com/mrajashree/Documents/blob/master/images/Rancher-Attributes.png)

2. Generate private key and certificate for your server

`openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -keyout privateKey.key -out certificate.crt`
The above command can be used for that

3. Upload/paste your private key and certificate in these fields. Upload the file PingIdP_metadata.xml in the field for `Metadata XML`

4. Click on `Save`. This saves the entire config in cattle db and generates Rancher Service Provider's metadata.
Download this metadata from http://x.x.x.x:8080/v1-auth/saml/metadata. Rename this file to RancherSP_metadata

<h3> Creating SP connection from PingFederate server </h3>

1. In PingFederate server, go to IdP Configuration -> SP CONNECTIONS -> Create New

2. Leave the Connection Type and Connection Options sections as they are by default. In the Import Metadata section, select FILE option and upload RancherSP_metadata file

3. The next section, `General Info` will have the following fields prefilled from the Rancher metadata you just uploaded:

PARTNER'S ENTITY ID -> http://x.x.x.x:8080/v1-auth/saml/metadata

CONNECTION NAME -> http://x.x.x.x:8080/v1-auth/saml/metadata

BASE URL -> http://x.x.x.x:8080

4. In Browser SSO, IDP-INITATED SSO and SP-INITIATED SSO have been selected as SAML Profiles. 

5. Assertion Lifetime is left 5 minutes (default value)

6. In the next section of `Assertion Creation`, click `Configure Assertion Creation`

	i) In Identity Mapping, select STANDARD option
	
	ii) Under Attribute Contract, SAML_SUBJECT is already provided by default. Over here we'll Extend the Contract by adding the attributes we provide to Rancher while configuring access control. Based on the example in previous section
	![Contract](https://github.com/mrajashree/Documents/blob/master/images/Attribute-Contract-SP%20connection.png)
	
	iii) In Authentication Source Mapping, click on 'Map New Adapter Instance'. Select 'PingOne HTML Form Adapter' (provided by default) and click Next
	
	iv) In Mapping Method, select the second option, i.e 
	`RETRIEVE ADDITIONAL ATTRIBUTES FROM A DATA STORE -- INCLUDES OPTIONS TO USE ALTERNATE DATA STORES AND/OR A FAILSAFE MAPPING`
	
	v) Under Attribute Sources & User Lookup, click on `Add Attribute Source`. Since we're using LDAP in this doc, select the already added LDAP server information from the dropdown. If you want to use a separate Data Store, click on `Manage Data Stores`. Once Data Store is selected click next
	
	vi) Under LDAP Directory Search, enter BASE DN, let SEARCH SCOPE be Subtree. In the section `Attributes to return from search`, you will see Subject DN is already listed. This is where we select our attributes from the LDAP data store. Select and add displayName, givenName, cn and memberOf (with Nested Groups option). Provide a search filter, for example, (sAMAccountName=${username}). 
	
	vii) Under Attribute Contract Fulfillment is where you map the attributes you specified while defining attribute contract in step 6.ii), to the attributes you selected from LDAP store in step 6.vi)
	
	This is how it should be mapped
	![mapping](https://github.com/mrajashree/Documents/blob/master/images/AttributeContractFulfillment.png)
	
	viii) Under Failsafe Attribute Source, select ABORT THE SSO TRANSACTION.
	
7. In Protocol Settings, click on `Configure Protocol Settings`
	i) Assertion Consumer Service URL should be http://x.x.x.x:8080/v1-auth/saml/acs (POST)
	
	ii) In Allowable SAML Bindings, select POST, REDIRECT and SOAP. Complete rest of the configuration for Protocol Settings
	
	iii) Configure the Credentials and Signature Verification sections
	
8. After saving this, when redirected to Activation & Summary page, set Connection Status to Active and click Save

<h2> Rancher UI </h2>
Back to Rancher UI, click on `Test`. This will provide you the login form, once authenticated, access control will be enabled

