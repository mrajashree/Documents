<h1> PingFederate-Rancher Configuration </h1>
The configuration involves following steps
1. Generating Ping IdP metadata from pingFederate server
2. Generating Rancher SP metadata from Rancher server
3. Creating SP Connection from pingFederate server to the Rancher server
4. Authenticating the user in Rancher

## Setting up Data Store on Ping
1. After installing PingFederate, select the option to configure PingFederate as an Identity Provider.

2. Go to **Server Configuration**. Click on **System Settings** -> **Data Store**s. In the Manage Data Stores page, click on **Add New Data Store**. 

	a. Data Store Type: In our example, we'll use the **LDAP** type. Select **LDAP** and click on **Next**. 
	
	b. LDAP Configuration: Provide details regarding your LDAP connection. The configuration will require the following infromation: HOSTNAME, LDAP TYPE (Active Directory in this doc), USER DN, PASSWORD. When you click on **Next**, it will attempt to connect to your LDAP server. 

## Generating Identitiy Provider Metadata 

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

	d. Signing Key: Select the certificate from the dropdown menu. Click on **Next**. 
	
	e. Metadata Signing: Select the signing certificate from the dropdown menu. Make sure to select the checkbox for **Include this certificate's public key certificate in the `<keyinfo>` element.** Select the checkbox for the **Include the Raw Key in the Signature `<keyvalue>` element.** Click on **Next**.
	
	f. XML Encryption Certificate: Select the Encryption certificate from the dropdown menu. Click on **Next**. 
	
	g. Export & Summary: Click on the **Export** button and save the file as `PingIdP_metadata.xml`. Click on **Done**.


## Generating Service Provider (Rancher) Metadata 

Before turning on access control, make sure the host registration URL is set correctly. Go to **Admin** -> **Settings**. Under **Host Registration URL**, verify the address is correct and click **Save**. If this is a fresh setup of Rancher and this URL has not been saved, the metadata will be generated incorrectly. 

> Note: If you have used Rancher before to add hosts, this field will have been saved the first time you went to add a host. 

1. On the instance running the Rancher server container, you will need to generate a private key and certificate. The following command can be used on the instance to generate a private key and certificate for your server.

```
openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -keyout privateKey.key -out certificate.crt
```

2. Save the private key and certificate details so that it can be uploaded to the UI for when you need to configure Shibboleth.

3. Go to **Admin** -> **Access Control**. Select **Shibboleth**. Enter the first four fields based on your attribute contract that was created earlier. 

	a. Display Name Field: `displayName`</br>
	b. User Name Field: `givenName`</br>
	c. UID Field: `cn`</br>
	d. Groups Field: `memberOf`</br>
	
![Rancher Access Control configuration attributes](https://github.com/mrajashree/Documents/blob/master/images/Rancher-Attributes.png)



4. Upload or paste your private key and certificate generated from step 1 into the fields. Upload the file `PingIdP_metadata.xml` that was created in PingFederate into the **Metadata XML** field.

4. Click on **Save**. This saves the entire configuration into the database and generates the Rancher Service Provider's metadata. 

5. Download the metadata file from http://<SERVER_IP>:8080/v1-auth/saml/metadata. Save the file as `RancherSP_metadata`.

## Creating a SP connection from PingFederate server 

1. In PingFederate server, go to **IdP Configuration**. Under **SP Connections**, click on **Create New**. 
	
	a. Connection Type: Keep the default options and click on **Next**. </br>
	b. Connection Options: Keep the default options and click on **Next**. </br>
	c. Import Metadata: select the **FILE** option and updload the `RancherSP_metadata` file that was saved from the Rancher setup. </br>
	d. General Info: Specific fields will be pre-filled from the Rancher metadata that was uploaded: 
	
		* PARTNER'S ENTITY ID -> http://<SERVER_IP>:8080/v1-auth/saml/metadata
		* CONNECTION NAME -> http://<SERVER_IP>:8080/v1-auth/saml/metadata
		* BASE URL -> http://<SERVER_IP>:8080

	e. Browser SSO: Click on **Configure Browser SSO**.</br>
	
		*  SAML Profiles: Select **IDP-INITATED SSO** and **SP-INITIATED SSO**.  Click on **Next**.</br>
		*  Assertion Lifetime: Keep the default values, which is 5 minutes. Click on **Next**. </br>
		*  Assertion Creation: Click on **Configure Assertion Creation**.</br>
	
			i) Identity Mapping: Select the **Standard** option and click on **Next**. 
			ii) Attribute Contract: The **SAML_SUBJECT** is already provided by default. We will extend the contract by adding the attributes we had provided to Rancher while configuring access control. For the Attribute Name Format, select `urn:mace:shibboleth:1.0:attributeNamespace:uri`. Here are the fields:
		 	
				* cn
				* displayName
				* givenName
				* memberOf
	
			Example:
	![Contract](https://github.com/mrajashree/Documents/blob/master/images/Attribute-Contract-SP%20connection.png)
	
			iii) Authentication Source Mapping: Click on **Map New Adapter Instance**. 
				* Adapter Instance: Select 'PingOne HTML Form Adapter' from the dropdown menu. Click on **Next**. 
				* Mapping Method: Select the second option, i.e 
		**RETRIEVE ADDITIONAL ATTRIBUTES FROM A DATA STORE -- INCLUDES OPTIONS TO USE ALTERNATE DATA STORES AND/OR A FAILSAFE MAPPING**. 
				* Attribute Sources & User Lookup: Click on **Add Attribute Source**. 
					* Data Store: In the **Attribute Source Description**, add in `LDAP server`. Select the LDAP server information from the dropdown menu. If you want to use a separate Data Store, click on `Manage Data Stores`. Click on **Next**. 
					* LDAP Directory Search: Enter the **BASE DN**, leave **SEARCH SCOPE** as Subtree. In the section **Attributes to return from search**, you will see **Subject DN** is already listed. We will need to add attributes from the LDAP data store. Under Root Object Class, select the `<Show All Attributes>`. Select each attribute and add all 4 required attributes., i.e `displayName`, `givenName`, `cn` and `memberOf` (with Nested Groups option). Click on **Next**. 
					* LDAP Filter: Provide a search filter, e.g. `sAMAccountName=${username}`. Click on **Next**. 
					* Attribute Contract Fulfillment: Map the attributes that were defined earlier to the attributes in the LDAP store. Click on **Next**.
				
		This is how the mapping should be done
	![mapping](https://github.com/mrajashree/Documents/blob/master/images/AttributeContractFulfillment.png)
					
					* Summary: Click on **Done**. 
				* Click on **Next** to move from **Attribute Sources & User Lookup** to **Failsafe Attribute Source**.
				* Failsafe Attribute Source: Select **ABORT THE SSO TRANSACTION**. Click on **Next**. 
				* Summary: Click on **Done**. 
			iv) Click on **Next** to move from **Authentication Source Mapping** to **Summary**. 
			v) Summmary: Click on **Done**. 
	
		* Click on **Next** to move from **Assertion Creation** to **Protocol Settings**. 
		* Protocol Settings: Click on **Configure Protocol Settings**. 
			i) Assertion Consumer Service URL:  Confirm that the Default is selected, the index is at `1` and the binding is `POST`. Edit the **Endpoint URL** so that it includes the server IP and port. The Endpoint URL should look like `http://<SERVER_IP>:8080/v1-auth/saml/acs`. Click on **Add** and then **Next**. 
			ii) Allowable SAML Bindings: De-select `ARTIFACT`, but ensure that `POST` `REDIRECT` and `SOAP` are selected. 
			iii) Signature Policy and Encryption Policy: This is dependent on the user but is not required. 
			If the user selects `Always sign the SAML Assertion` in Signature Policy, then the user should select **Encrypt Entire Assertion** in the Encryption Policy.
			iv) Summary: Click on **Done**. 
	
		* Click on **Next** to move from **Protocol Settings** to **Summary**. 
		* Summary: Click on **Done**. 
	f. Click on **Next** to move from **Browser SSO** to **Credentials**. </br>
	g. Credentials: Click on **Configure Credentials**. </br>
		
		* Back-Channel Authentication: Click on **Configure**.</br>
			i) Inbound Authentication Type: Select **Digital Signature (Browser SSO Profile Only)**. Keep **Require SSL** selected. Click on **Next**. </br>
			ii) Summary: Click on **Done**. </br>
		* Click on **Next** to move from **Back-Channel Authentication** to **Digital Signature Settings**.</br>
		* Digital Signature Settings: Select the signing certificate from the dropdown menu. Make sure to select the checkbox for **Include this certificate's public key certificate in the <keyinfo> element.** Select the checkbox for the **Include the Raw Key in the Signature <keyvalue> element.** Click on **Next**. </br>
		* Signature Verification Settings: Click on **Manage Signature Verification Settings**. </br>
			i) Trust Model: Leave default selection, i.e. `UNANCHORED`. Click on **Next**.</br>
			ii) Signature Verification Certificate: Select the certificate from the dropdown for the **Primary** and click on **Next**. </br>
			iii) Summary: Click on **Done**. </br>
		* Click on **Next** to move from **Signature Verification Settings** to **Summary**. </br>
		* Summary: Click on **Done**. </br>
	h. Click on **Next** to move from **Credentials** to **Activation & Summary**. 
	
	g. Activation & Summary: Set Connection Status to `Active` and click on **Save**.

## Activating Access Control in Rancher

In the Rancher UI on the Shibboleth page, click on **Test**. A browser will pop-up to provide you the login form for the LDAP server. Once authenticated, access control will be enabled for Rancher.

