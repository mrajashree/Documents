<h1> PingFederate Configuration </h1>

1. After installing PingFederate, select the option to configure PingFederate as an Identity Provider.

2. Go to Server Configuration -> SYSTEM SETTINGS -> Data Stores -> Add New Data Store. It has options for DATABASE, LDAP and CUSTOM. This document uses LDAP store to go through the configuration. Provide details such as HOSTNAME, LDAP TYPE (Active Directory in this doc), USER DN, PASSWORD

<h2> In case SP connection is not created </h2>
<h3> Generating Identitiy Provider metadata </h3>

1. Go To Server Configuration -> ADMINISTRATIVE FUNCTIONS -> Metadata Export -> Select Information to include in Metadata manually -> Protocol preselected -> Attribute Contract.
These attributes are the ones to be provided as input for Rancher Access Control configuration.

For example this is the Attribute contract
![Attribute Contract IdP](https://github.com/mrajashree/Documents/blob/master/images/IdP-metadata-creation.png)

and the corresponding fields for Rancher Access Control configuration attributes
![Rancher Access Control configuration attributes](https://github.com/mrajashree/Documents/blob/master/images/Rancher-Attributes.png)

2. Complete the rest of the steps to generate metadata, export it and save as PingIdP_metadata.xml

<h3> Generating Service Provider (Rancher) metadata </h3>

Before proceeding with these steps, make sure the host setting is saved correctly. Go to Admin -> Settings -> Host Registration URL -> Choose 'This site's address' and hit Save

1. On Rancher UI, enter the first four fields under Shibboleth access control based on your attribute contract from previous section. (Refer image *Rancher Access Control configuration attributes* in previous section point 1)

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


