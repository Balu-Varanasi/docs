---
id: saml-authentication
title: SAML Authentication
---

## Intro

There are three steps in configuring a SAML Authentication using Microsoft Entra ID.

- Step 1: Preparing the Environment and Configuring Django-allauth
- Step 2: Register Your Application with Microsoft Entra ID
- Step 3: Configure Your Django Application


## Step 1: Preparing the Environment and Configuring Django-allauth

- a) Ubuntu Server Setup
- b) Django-allauth Configuration
- c) Configure Django to Use HTTPS Only
- d) Proxy Configuration for HTTPS 

#### a) Ubuntu Server Setup

The initial setup on an Ubuntu server involves installing necessary libraries and packages to support Secure Assertion Markup Language (SAML) authentication and secure communications. Here's how to do it:

1. **Install Required Native Libraries**: SAML integration requires specific native libraries related to XML processing and security. These libraries are essential for parsing and handling SAML assertions securely. On an Ubuntu server, execute the following command to install these libraries:
   ```bash
   sudo apt-get update
   sudo apt-get install libxml2-dev libxmlsec1-dev libxmlsec1-openssl
   ```
   This command installs the `libxml2-dev` library for XML processing, `libxmlsec1-dev` for XML security operations, and `libxmlsec1-openssl` for OpenSSL support in XML security operations.

2. **Install xmlsec via pip**: The `xmlsec` Python package is a wrapper around the XMLSec library, facilitating XML signature and encryption in Python. After installing the required native libraries, install `xmlsec` by running:
   ```bash
   pip install xmlsec
   ```
   This step is crucial for enabling your Django application to handle SAML assertions securely.

3. **Install django-allauth with SAML Support**: To integrate Microsoft Entra ID using django-allauth, you need to install the `django-allauth` package with SAML extensions. This allows your Django application to authenticate users via SAML tokens issued by Microsoft Entra. Use Poetry to add `django-allauth` with SAML support to your project:
   ```bash
   poetry add "django-allauth[saml]"
   ```
   This command ensures that `django-allauth` and its dependencies are installed and managed by Poetry, simplifying dependency management and project setup.

##### References:
- https://xmlsec.readthedocs.io/en/stable/install.html

#### b) Django-allauth Configuration

Once the necessary packages are installed, configure your Django project to use django-allauth with SAML:

1. **Update INSTALLED_APPS**: Ensure your Django `settings.py` includes `django-allauth` and its SAML provider by adding them to the `INSTALLED_APPS` section:
   ```python
   INSTALLED_APPS = [
       # Other installed apps
       "allauth",
       "allauth.account",
       "allauth.socialaccount",
       "allauth.socialaccount.providers.saml",
   ]
   ```
   This step enables django-allauth in your project and adds support for SAML-based authentication.

#### c) Configure Django to Use HTTPS Only

For security reasons, especially when handling authentication tokens, configure Django to operate under HTTPS only:

1. **Set SECURE_PROXY_SSL_HEADER**: In your `settings.py`, add the following configuration:
   ```python
   SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")
   ```
   This tells Django to use the `HTTP_X_FORWARDED_PROTO` header to determine if the request is secure (HTTPS). This is important when deploying Django behind a proxy server like Nginx.

#### d) Proxy Configuration for HTTPS

Ensure that your proxy server (e.g., Nginx) is configured to pass the correct headers to Django:

1. **Configure Nginx**: Add the following line to your Nginx site configuration to ensure the `HTTP_X_FORWARDED_PROTO` header is passed correctly to Django:
   ```nginx
   proxy_set_header X-Forwarded-Proto $scheme;
   ```
   This configuration ensures Django can accurately determine the request scheme (HTTP or HTTPS), which is essential for secure operation and proper URL construction.


## Step 2: Register Your Application with Microsoft Entra ID
Follow the instructions in the official [Quickstart](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/add-application-portal) guide to configure an `Enterprise Application` and register it in the `App registrations` section.

After configuring an `Enterprise application` and registering it under `App registrations`, obtain the `Display name` and `Directory (tenant) ID` from the `App registration` dashboard. These details will later serve as the `SAML_APP_NAME` and `SAML_APP_TENANT_ID` environment variables.

To set up `Single Sign-On` values, go to `Home > Default Directory > Manage > Enterprise applications > SAML_APP_NAME > Manage > Single sign-on`. Follow the instructions in the image below to configure endpoints. Remember to replace `test-org-inc` in the URLs with the slug you plan to use; this slug will be referenced as the `SAML_APP_CLIENT_ID` environment variable later.

![SSO - SAML - Microsoft Entra ID Configuration](/img/microsoft-entra-id-saml-sso.png)

When configuring the SAML signing certificate (as seen in the third section of the image above), you'll come across options to determine how SAML tokens should be signed and which algorithm should be used. Here are the key details for each option:

1. **Signing Option**: This setting determines which parts of the SAML tokens provided by Microsoft Entra ID during the authentication process will be digitally signed. The options are:
   - **Sign SAML response**: This option signs the entire SAML response that includes the assertion. It ensures the integrity of the response from Microsoft Entra ID but not the assertion itself.
   - **Sign SAML assertion**: This option signs just the assertion within the SAML response. The assertion contains the authentication and authorization data, and signing it ensures its integrity.
   - **Sign SAML response and assertion**: This option provides the highest level of security by signing both the entire response and the assertion within it. It is generally the recommended setting as it ensures the integrity of both the response and the contained assertion.

2. **Signing Algorithm**: This setting specifies the cryptographic hashing algorithm used to sign the SAML tokens. The options are:
   - **SHA-1**: An older hashing algorithm that is faster but considered less secure due to vulnerabilities that have been discovered over time.
   - **SHA-256**: A more secure hashing algorithm that is part of the SHA-2 family. It provides a longer hash value and is designed to counteract the vulnerabilities found in SHA-1. It is recommended to use SHA-256 for better security.

### Recommendations:

- **Signing Option**: Choose to "Sign SAML response and assertion" for the highest security level, ensuring that both the response and the assertion are verified as coming from a trusted source without tampering.

- **Signing Algorithm**: Opt for SHA-256, given its enhanced security properties over SHA-1. This is especially important for applications that handle sensitive information or operate in environments where security is a primary concern.

After configuring the certificate, download the `Certificate (Base64)` and save it. This will be used as the `SAML_APP_CERTIFICATE` environment variable later.

#### References:
1. [SAML authentication with Microsoft Entra ID](https://learn.microsoft.com/en-us/entra/architecture/auth-saml)
2. [Quickstart: Add an enterprise application](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/add-application-portal)
3. [Quickstart: Create and assign a user account](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/add-application-portal-assign-users)
4. [What is single sign-on in Microsoft Entra ID?](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/what-is-single-sign-on)
5. [Plan a single sign-on deployment](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/plan-sso-deployment)

## Step 3: Configure Your Django Application
To configure your Django project for SAML authentication with Microsoft Entra ID, you will need to update your `settings.py` file with specific settings that define how your application interacts with Microsoft's identity services. 

Configure your application's SAML settings using environment variables for better security and flexibility. This approach allows you to change settings without modifying your codebase directly. 

Below are sample enviornmental variables:

   ```bash
export SAML_APP_NAME="Test Org Inc"
export SAML_APP_CLIENT_ID="test-org-inc"
export SAML_APP_TENANT_ID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
export SAML_APP_CERTIFICATE="
-----BEGIN CERTIFICATE-----
MIIC8...
-----END CERTIFICATE-----
"   
   ```

Then, add the following configuration in your `settings.py`:
   ```python
   # Microsoft Entra ID SAML Configuration
   SAML_APP_NAME = os.getenv("SAML_APP_NAME", "")
   SAML_APP_CLIENT_ID = os.getenv("SAML_APP_CLIENT_ID", "")
   SAML_APP_TENANT_ID = os.getenv("SAML_APP_TENANT_ID", "")
   SAML_APP_CERTIFICATE = os.getenv("SAML_APP_CERTIFICATE", "")

   SAML_APP = {
       "name": SAML_APP_NAME,
       "provider_id": f"https://login.microsoftonline.com/{SAML_APP_TENANT_ID}/",
       "client_id": SAML_APP_CLIENT_ID,
       "settings": {
           "attribute_mapping": {
               "uid": "http://schemas.microsoft.com/identity/claims/objectidentifier",
               "email": "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress",
               "first_name": "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/givenname",
               "last_name": "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/surname",
           },
           "idp": {
               "entity_id": f"https://sts.windows.net/{SAML_APP_TENANT_ID}/",
               "sso_url": f"https://login.microsoftonline.com/{SAML_APP_TENANT_ID}/saml2",
               "slo_url": f"https://login.microsoftonline.com/{SAML_APP_TENANT_ID}/saml2",
               "x509cert": SAML_APP_CERTIFICATE,
           },

            # "advanced": {
            #     "strict": False,
            #     "authn_requests_signed": False,
            #     "logout_request_signed": False,
            #     "logout_response_signed": False,
            #     "requested_authn_context": False,
            #     "sign_metadata": False,
            #     "want_assertion_encrypted": False,
            #     "want_assertion_signed": True,
            #     "want_messages_signed": False,
            # },
       },
   }

   SOCIALACCOUNT_PROVIDERS = {
       "saml": {
           "APPS": [SAML_APP],
       },
   }
   ```
   This configuration specifies your application's name, client ID, tenant ID, and certificate directly from environment variables, providing a secure and flexible way to manage your SAML integration. The `attribute_mapping` and `idp` configurations define how user attributes are mapped from SAML assertions and detail the identity provider (IdP) settings necessary for SAML assertions.

   #### Handling `metadata_url` and `x509cert` in django-allauth SAML Configuration
   Avoid specifying `metadata_url` and `x509cert` together in django-allauth's SAML `idp` settings. If both are provided, allauth prioritizes the `idp` settings fetched from `metadata_url`, ignoring the manually set `x509cert`. Despite examples in the official documentation, using both can lead to confusion.

   ##### References:
   1. https://docs.allauth.org/en/latest/socialaccount/providers/saml.html
   2. https://github.com/pennersr/django-allauth/blob/main/allauth/socialaccount/providers/saml/utils.py#L135   

   #### Configuring `advanced` SAML app settings

   While Setting up Single Sign-On with SAML in step 2, if you choose to sign the SAML assertion or both the SAML response and assertion in Microsoft Entra ID, then configuring the `advanced` settings in your Django application's SAML configuration (as part of django-allauth) to require signed assertions is necessary. This ensures that your Django application validates the integrity of the received SAML assertions according to the signing preferences set in Microsoft Entra ID.

   By setting `"want_assertion_signed": True`, you are instructing your Django application to expect and validate the signature of the SAML assertion. This is an important security measure, as it ensures that the assertions your application processes are indeed sent by Microsoft Entra ID and have not been tampered with during transmission.

   Here's how the updated configuration would look, including the `advanced` settings to enforce assertion signing:

   ```python
   SAML_APP = {
      ...
      "settings": {
         ...
         "advanced": {
            "want_assertion_signed": True,
         },
      },
   }
   ```

   This approach ensures that your application aligns with the security configurations set on the identity provider side (Microsoft Entra ID), maintaining the integrity and trustworthiness of the authentication process.

   ##### References:
   1. https://github.com/pennersr/django-allauth/blob/main/allauth/socialaccount/providers/saml/utils.py#L108
   2. https://github.com/SAML-Toolkits/python3-saml/blob/master/src/onelogin/saml2/response.py#L312