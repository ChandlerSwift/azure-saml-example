# Azure Active Directory SAML sample code/configuration

This example is based off the example code in
[crewjam/saml](https://github.com/crewjam/saml), but tweaked to work with
Microsoft's Azure Active Directory SAML authentication.

### Generating a Certificate Pair
We use roughly the same command as in the upstream example, but use a `.crt`
extension rather than `.cert`, since Azure's certificate upload process doesn't
recognize `.cert` files. The common name is not used as part of the
authentication process, so can be anything.

```sh
openssl req -x509 \
    -newkey rsa:2048 \
    -keyout myservice.key \
    -out myservice.crt \
    -days 365 \
    -nodes \
    -subj "/CN=myservice.example.com"
```

### Setting up Auth with an Azure Active Directory Application
Unlike SAMLtest, Azure Active Directory doesn't accept a metadata upload, so we
need to take a few extra manual steps in the Portal.

1. Generate a certificate pair as above. Ensure your application is set to use
   the generated files.
2. In the Azure Portal, go to Azure Active Directory, and select "App
   registrations" from the menu on the left. Click "New Registration". Choose a
   name. This example is written as a single tenant application. Under Redirect
   URI, select "Web" and enter "http://localhost:8000/saml/acs". Click
   "Register".
3. Copy the "Application (client) ID" from the main page of the app registration
   you've created. Put this where the code sample says
   `<your application's Client ID>`.
4. From "Endpoints", copy the "Federation metadata document" URL. It should be
   `https://login.microsoftonline.com/<your tenant UUID>/federationmetadata/2007-06/federationmetadata.xml`.
   Put this on line 29 for the `idpMetadataURL`'s value. (This is what your
   application will use to discover the endpoints for other functions.)
5. On the "Certificates & secrets" page, Select the "Certificates" tab and
   upload the certificate from step 1.
6. Compile and run!

### Notable changes
Without the `EntityID` (which defaults to the empty string), the following error
is seen:
```
AADSTS700016: Application with identifier 'http://localhost:8000/saml/metadata' was not found in the directory 'mydirectory'. This can happen if the application has not been installed by the administrator of the tenant or consented to by any user in the tenant. You may have sent your authentication request to the wrong tenant.
```

If the `EntityID` is simply set to the UUID (without the `spn:` prefix), the
request succeeds, but the SAML library rejects the response as invalid, because
the response, which is compared to the `EntityID`, _does_ contain the prefix.
```
assertion invalid: assertion Conditions AudienceRestriction does not contain "<your application's UUID>"
```

(This comparison is made
[in `service_provider.go`](https://github.com/crewjam/saml/blob/main/service_provider.go#L985).)

The response that Microsoft returns doesn't have a `cn` attribute; instead, we
use the `http://schemas.microsoft.com/identity/claims/displayname` attribute.

### Caveats
We notably didn't upload our certificate. Given this use case (this application
doesn't make actual requests once we have the user's name and email address),
this isn't a security concern, but I'm not sure it's safe in all cases. (Though
I have only an afternoon's understanding of SAML, so don't take my word for it!)
