PHP OpenID Connect and UMA Basic Client
========================
A simple library that allows an application to authenticate a user through the basic OpenID Connect flow in addition to  User Managed Access (UMA) 2.0.
This library hopes to encourage OpenID Connect and User Managed Access use by making it simple enough for a developer with little knowledge of
the OpenID Connect and UMA 2.0 protocols to setup authentication and resource protection.

A special thanks first to Michael Jett with the initial [PHP OpenID Connect libary](https://github.com/jumbojett/OpenID-Connect-PHP) form which this library was developed from.

# Requirements #
 1. PHP 5.4 or greater
 2. CURL extension
 3. JSON extension

## Install ##
 1. Install library using composer
```
composer require shihjay2/openid-connect-uma-php
```
 2. Include composer autoloader unless you are using [Laravel](https://github.com/laravel/laravel)
```php
require __DIR__ . '/vendor/autoload.php';
```

## Example 1: Dynamic Registration ##

```php
use Shihjay2\OpenIDConnectUMAClient;

$oidc = new OpenIDConnectClient("https://id.provider.com");

$oidc->register();
$client_id = $oidc->getClientID();
$client_secret = $oidc->getClientSecret();

// Be sure to add logic to store the client id and client secret
```

## Example 2: Dynamic Registration to an UMA Server ##

```php
use Shihjay2\OpenIDConnectUMAClient;

$oidc = new OpenIDConnectClient("https://uma.provider.com");
// in case your resource has same domain as your UMA or OIDC server
$oidc->setSessionName('create_your_own_session_name');
$oidc->addRedirectURLs('https://client.url1.com');
$oidc->addRedirectURLs('https://client.url2.com');
$oidc->addRedirectURLs('https://client.url3.com');
$oidc->addRedirectURLs('https://client.url4.com');
$oidc->addScope('openid');
$oidc->addScope('email');
$oidc->addScope('profile');
$oidc->addScope('address');
$oidc->addScope('phone');
$oidc->addScope('offline_access');
$oidc->addScope('uma_authorization');

// If you plan to register your client as an UMA resource, set uma_protection as a scope

$oidc->addScope('uma_protection');
$oidc->setUMA(true);
$oidc->register();
$client_id = $oidc->getClientID();
$client_secret = $oidc->getClientSecret();

// Be sure to add logic to store the client id and client secret
```

## Example 3: Basic OpenID Connect Client ##

```php
use Shihjay2\OpenIDConnectUMAClient;

$oidc = new OpenIDConnectClient('https://id.provider.com',
                                'ClientIDHere',
                                'ClientSecretHere');
$oidc->setCertPath('/path/to/my.cert');
$oidc->authenticate();
$name = $oidc->requestUserInfo('given_name');

```
[See openid spec for available user attributes][1]

## Example 4: Connecting to an UMA server as a resource server and register resources ##
```php
use Shihjay2\OpenIDConnectUMAClient;

$oidc = new OpenIDConnectClient('https://uma.provider.com',
                                'ClientIDHere',
                                'ClientSecretHere');
$oidc->setCertPath('/path/to/my.cert');
$oidc->setRedirectURL('https://resource.url.com');
$oidc->setSessionName('create_your_own_session_name');
$oidc->setUMA(true);
$oidc->setUMAType('resource_server');
$oidc->authenticate();
// This is the Protection Access Token (PAT) - this is saved in the next call to register resources
$pat = $oidc->getAccessToken();
// Save your refresh token in your database
$refresh_token = $oidc->getRefreshToken();
// Register resource sets
$resource_set_array[] = [
    'name' => 'Resource 1',
    'icon' => 'https://icon1.png',
    'scopes' => [
        'https://resource.url.com/resource1',
        'view',
        'edit'
    ]
];
$resource_set_array[] = [
    'name' => 'Resource 2',
    'icon' => 'https://icon2.png',
    'scopes' => [
        'https://resource.url.com/resource2',
        'view',
        'edit'
    ]
];
$resource_set_array[] = [
    'name' => 'Resource 3',
    'icon' => 'https://icon3.png',
    'scopes' => [
        'https://resource.url.com/resource3',
        'view',
        'edit'
    ]
];
foreach ($resource_set_array as $resource_set_item) {
    $response = $oidc->resource_set($resource_set_item['name'], $resource_set_item['icon'], $resource_set_item['scopes']);
    if (isset($response['resource_set_id'])) {
        // Success!
        $resource_set_id = $response['resource_set_id'];
        $user_access_policy_uri = $response['user_access_policy_uri'];
        // Save the resource set ID and user access policy URI in your database
        // Also a good idea to save the scopes of each resource
    }
}
```
[See UMA spec for obtaining the PAT and registering resources][4]

## Example 5: Connecting to an UMA server as a client ##
```php
use Shihjay2\OpenIDConnectUMAClient;

$oidc = new OpenIDConnectUMAClient('https://uma.provider.com',
                                'ClientIDHere',
                                'ClientSecretHere');
$oidc->setSessionName('create_your_own_session_name');
$oidc->setUMA(true);
$oidc->setUMAType('client');
$oidc->authenticate();
$access_token = $oidc->getAccessToken();
// Save this access token as a session for future calls such as Example 7
```
[See UMA spec for access protected resource without a token][5]

## Example 6: Requesting Party Token request in UMA 2.0 flow, Step 1 ##
```php
use Shihjay2\OpenIDConnectUMAClient;

// Permission ticket received when initially making a call to the resource without a Requesting Party Token (RPT)
$permission_ticket = 'permission_ticket'
$oidc = new OpenIDConnectUMAClient('https://uma.provider.com',
                                'ClientIDHere',
                                'ClientSecretHere');
$oidc->setRedirectURL('https://client.url.com');
$oidc->rqp_claims($permission_ticket);
// You'll be then redirected to the UMA server for claims gathering...
```
[See UMA spec for claims gathering][2]

## Example 7: Requesting Party Token request in UMA 2.0 flow, Step 2 ##
```php
use Shihjay2\OpenIDConnectUMAClient;

$oidc = new OpenIDConnectUMAClient('https://uma.provider.com',
                                'ClientIDHere',
                                'ClientSecretHere');
$oidc->setSessionName('create_your_own_session_name');
// Access token from Example 5
$oidc->setAccessToken($access_token);
$oidc->setRedirectURL($url);
$result = $oidc->rpt_request($permission_ticket);
// If claims gathering successful, RPT will be granted by the UMA server
$rpt = $result['access_token'];
```
[See UMA spec for RPT request][3]

## Example 9: Resource server requests permissions on client's behalf with the UMA server ##
```php
use Shihjay2\OpenIDConnectUMAClient;

$oidc = new OpenIDConnectUMAClient('https://uma.provider.com',
                                'ClientIDHere',
                                'ClientSecretHere');
$oidc->setUMA(true);
// Resource set ID taken from Example 4
// Scopes as an array, taken from Example 4
$resource_set_id = 'resource_set_id';
$scopes = array('scope1', 'scope2');
$permission_ticket = $oidc->permission_request($resource_set_id, $scopes);
// Send permission ticket back as a WWW-Authticate header back to client
```
[See UMA spec for permission tickets][6]

## Example 10: Resource server determines RPT status ###
```php
use Shihjay2\OpenIDConnectUMAClient;

$oidc = new OpenIDConnectUMAClient('https://uma.provider.com',
                                'ClientIDHere',
                                'ClientSecretHere');
$oidc->setUMA(true);
// RPT comes from Authorization: Bearer request header, in Laravel, get it like this:
$payload = $request->header('Authorization');
if ($payload) {
    // RPT, Perform Token Introspection
    $rpt = str_replace('Bearer ', '', $payload);
} else {
    // Go back to Example 9 to get permission ticket
}
$result = $oidc->introspect($rpt);
if ($result['active'] == false) {
    // respond back to client why the RPT status failed
} else {
    // redirect to the resource
}
```
[See UMA spec for introspection][7]

## Example 11: Network and Security ##
```php
// Configure a proxy
$oidc->setHttpProxy("http://my.proxy.com:80/");

// Configure a cert
$oidc->setCertPath("/path/to/my.cert");
```

## Example 12: Request Client Credentials Token ##

```php
use Shihjay2\OpenIDConnectUMAClient;

$oidc = new OpenIDConnectUMAClient('https://id.provider.com',
                                'ClientIDHere',
                                'ClientSecretHere');
$oidc->providerConfigParam(array('token_endpoint'=>'https://id.provider.com/connect/token'));
$oidc->addScope('my_scope');

// this assumes success (to validate check if the access_token property is there and a valid JWT) :
$clientCredentialsToken = $oidc->requestClientCredentialsToken()->access_token;

```

## Example 13: Request Resource Owners Token (with client auth) ##

```php
use Shihjay2\OpenIDConnectUMAClient;

$oidc = new OpenIDConnectUMAClient('https://id.provider.com',
                                'ClientIDHere',
                                'ClientSecretHere');
$oidc->providerConfigParam(array('token_endpoint'=>'https://id.provider.com/connect/token'));
$oidc->addScope('my_scope');

//Add username and password
$oidc->addAuthParam(array('username'=>'<Username>'));
$oidc->addAuthParam(array('password'=>'<Password>'));

//Perform the auth and return the token (to validate check if the access_token property is there and a valid JWT) :
$token = $oidc->requestResourceOwnerToken(TRUE)->access_token;

```

## Example 14: Basic client for implicit flow e.g. with Azure AD B2C (see http://openid.net/specs/openid-connect-core-1_0.html#ImplicitFlowAuth) ##

```php
use Shihjay2\OpenIDConnectUMAClient;

$oidc = new OpenIDConnectUMAClient('https://id.provider.com',
                                'ClientIDHere',
                                'ClientSecretHere');
$oidc->setResponseTypes(array('id_token'));
$oidc->addScope(array('openid'));
$oidc->setAllowImplicitFlow(true);
$oidc->addAuthParam(array('response_mode' => 'form_post'));
$oidc->setCertPath('/path/to/my.cert');
$oidc->authenticate();
$sub = $oidc->getVerifiedClaims('sub');

```


## Development Environments ##
In some cases you may need to disable SSL security on on your development systems.
Note: This is not recommended on production systems.

```php
$oidc->setVerifyHost(false);
$oidc->setVerifyPeer(false);
```

  [1]: http://openid.net/specs/openid-connect-basic-1_0-15.html#id_res

  [2]:
  https://docs.kantarainitiative.org/uma/ed/uma-core-2.0-19.html#claim-redirect

  [3]:
  https://docs.kantarainitiative.org/uma/ed/uma-core-2.0-19.html#uma-grant-type

  [4]:
  https://docs.kantarainitiative.org/uma/ed/uma-core-2.0-19.html#rs-gets-pat

  [5]:
  https://docs.kantarainitiative.org/uma/ed/uma-core-2.0-19.html#client-attempts-tokenless-access

  [6]:
  https://docs.kantarainitiative.org/uma/ed/uma-core-2.0-19.html#register-permission

  [7]:
  https://docs.kantarainitiative.org/uma/ed/uma-core-2.0-19.html#check-rpt-status

## Contributing ###
 - All pull requests welome!
