![image](https://storage.googleapis.com/perimeterx-logos/primary_logo_red_cropped.png)

#

# [PerimeterX](http://www.perimeterx.com) F5 BIG-IP SDK

## Table of Contents

- [Dependencies](#dependencies)
- [Installation](#installation)
- [First-Party Integration](#firstParty)
- [Upgrading](#upgrading)
- [Basic Usage Example](#basic-usage)
- [Block Score Configuration](#score-config)
- [Configuration](#configuration)
  - [Directives](#directives)
  - [Additional Information](#additional_information)
  - [Enable Module Header](#enable_module_header)
  - [Test Block Flow on Monitoring Mode](#test_block_flow_on_monitoring_mode)
  - [Enrich Custom Parameters](#enrich_custom_parameters)
  - [JavaScript Sensor Injection](#inject_sensor)
  - [Upstream Score Header](#scoreHeader)
  - [Login Credentials Extraction](#loginCredentialsExtraction)
  - [User Identifiers](#userIdentifiers)
- [Configurational Classes](#configurational_classes)
- [Advanced Blocking Response](#advanced_blocking_response)
- [Troubleshooting](#troubleshooting)

## <a name="dependencies"></a> Dependencies

- F5 >= v11.5
- HSSR and HSSR-helper iRules ([F5 DevCentral](https://clouddocs.f5.com/api/irules/HTTP-Super-SIDEBAND-Requestor-Client-Handles-Redirects-Cookies-Chunked-Transfer-APM-Access-etc.html))

## <a name="installation"></a> Installation

This guide describes the step necessary to complete to install a fully functioning PerimeterX module on a F5 appliance

For access to the Enforcer's SDK, contact your PerimeterX Solution Architect or [PerimeterX Support](support@perimeterx.com)

### Configuring a virtual server and pool for PerimeterX backend requests

> Note: The BIG-IP device must have public internet access in order to communicate with the PerimeterX cloud services. Make sure to include the proper routing and gateway configuration.

Import the HSSR and HSSR-helper iRules downloaded from [F5 DevCentral](https://clouddocs.f5.com/api/irules/HTTP-Super-SIDEBAND-Requestor-Client-Handles-Redirects-Cookies-Chunked-Transfer-APM-Access-etc.html). These iRules are used for http client communication and TLS/SSL communication with PerimeterX backends.

![alt text](/screenshots/hssr.gif)
![alt text](/screenshots/hssr-helper.gif)

> Important Note: Make sure that the iRule names are: HSSR and HSSR-helper (corresponding to each rule).

### Configure Data Groups

1. Edit the `px-datagroup.txt` file and replace all instances of `APP_ID` with PerimeterX's app id.
2. In the same file, replace `AUTH_TOKEN` and `COOKIE_SECRET` with the correct PerimeterX values.
3. `ssh` to the F5 instance and run the content of `px-datagroup.txt` in the shell. This will create the configuration data group for the iRule.

### Configuring Pool: px_backend_pool

1. Under `Local Traffic` -> `Pools` -> `Pool List`, create a new pool.
2. Set the pool name to: `px_backend_pool`.
3. Set `Health Monitor` to tcp_half_open.
4. Select `new FQDN Node`.
5. Set `Node Name` to your app id.

> Note: App id can be retrieved from the PerimeterX portal under Admin->Applications.

6. Set `Address` to sapi-<APP_ID>.perimeterx.net
7. Set the `Service Port` to 443.
8. Set `Auto Populate` to Enabled.
9. Click `Add` & `Finished`

![alt text](/screenshots/px-backend_configure.gif)

### Configuring Virtual Server: px_backend_vip

1. Under `Local Traffic` -> `Virtual Servers` -> `Virtual Servers List`, create new virtual server. This virtual server must have external access for the pool members.
2. Set `Name` to `px_backend_<APP_ID>_vip`. The naming convention is important as the PerimeterX iRule uses this vip to send backend requests.

> note: Make sure to replace <APP_ID> with the same app id used in the previous section.

3. Set `Source Address` to 0.0.0.0/0
4. Set `Destination Address/Mask` Set any ip of a node that doesn't exist (for example: 10.0.0.30).
5. Set `Service Port` to a random, not publicly accessible port. (for example: port 55000).
6. Set `HTTP Profile` to http.
7. Configure the SSL Profile (Server) to use `serverssl`.
8. Configure the Source Address Translation to `Auto Map`.
9. Under `Resources` enable the `HSSR-helper` iRule.
10. Set `px_backend_pool` as the Default pool.

![alt text](/screenshots/vip_configure.gif)

### Configuring Activities Report

this step is crucial for the PerimeterX iRule to send statistics to PerimeterX backend and show data in the portal.
In order to send statistics and logs from the PerimeterX module in an asynchronous way, we will use Syslog.

> Note: PerimeterX backends are set to reject any unauthorized IP address, please contact your designated PerimeterX Solution Architect to authorize your backends IP address with PerimeterX backends.

#### Configuring an SSL Server Profile

1. Under `Local Traffic` -> `Profiles` -> `SSL` -> `Server` create a new profile.
2. Set `Name` to `px-syslog-ssl-profile`.
3. Set `Parent Profile` to `serverssl-insecure-compatible`.
4. Click `Finished`.

#### Configuring Pool: px_secure_syslog_pool

1. Under `Local Traffic` -> `Pools` -> `Pool List` create a new pool.
2. Set `Name` to `px_secure_syslog_pool`.
3. Set the node to `New FQDN Node`.
4. Set `Node Name` to `px_activities_node`.
5. Set `FQDN` to `px-fst-syslog.perimeterx.net`.
6. Set `Service Port` to `6514`.
7. Set `Auto Populate` to `Enabled`.
8. Click `Add` and `Finished`.

#### Configuring Virtual Server: px_syslog_vs

1. Under `Local Traffic` -> `Virtual Servers` -> `Virtual Servers List`, create new virtual server. This virtual server must have external access for the pool members.
2. Set `Name` to `px_syslog_vs`.
3. Set `Source Address` to 0.0.0.0/0
4. Set `Destination Address/Mask` Set any ip of a node that doesn't exist (for example: 10.0.0.20).
5. Set `Service Port` to 514.
6. Configure the SSL Profile (Server) to use `px-syslog-ssl-profile`.
7. Set `px_secure_syslog_pool` as the Default pool.
8. Click `Finished`.

#### Configuring Pool: px_syslog_pool

1. Under `Local Traffic` -> `Pools` -> `Pool List` create a new pool.
2. Set `Name` to `px_syslog_pool`. The naming convention is important as the PerimeterX iRule use this vip to send backend requests.
3. Set `Health Monitor` to `tcp_half_open`.
4. Set the node to `New Node`.
5. Set `Node Name` to `px_vs_syslog`.
6. Set `Address` to the same address as the `px_syslog_vs` virtual server (in the example above 10.0.0.20).
7. Set `Service Port` to 514.
8. Click `Add` and `Finished`.

#### Configure High Speed Login

1. Under `System` -> `Logs` -> `Configuration` -> `Log Destinations` create a new destination.
2. Set `Name` to `perimeterx_hsl`.
3. Set `Type` to `Remote-High-Speed Log`.
4. Set `Pool Name` to `px_syslog_pool`.
5. Set `Protocol` to `TCP`.
6. Click `Finished`.

![alt text](/screenshots/perimeterx_hsl_configure.gif)

#### Configure Syslog

1. Under `System` -> `Logs` -> `Configuration` -> `Log Destinations` create a new destination.
2. Set `Name` to `perimeterx_syslog`.
3. Set `Type` to `Remote Syslog`.
4. Set `Syslog Format` to `Syslog`
5. Set `Forward To` to `perimeterx_hsl`.
6. Click `Finished`.

![alt text](/screenshots/perimeterx_syslog_configure.gif)

#### Configure Publisher

1. Under `System` -> `Logs` -> `Configuration` -> `Log Publishers` create a new publisher.
2. Set `Name` to `perimeterx-publisher`.
3. Under `Destinations` move `perimeterx_syslog` to `Selected`. This will forward the logs to the hsl we previously configured.
4. Click `Finished`.

![alt text](/screenshots/perimeterx-publisher_configure.gif)

#### Configure Log Filters

1. Under `System` -> `Logs` -> `Configuration` -> `Log Filters` create a new filter.
2. Set `Name` to `perimeterx_filter`.
3. Set `Severity` to `Debug`.
4. Set `Source` to `all`.
5. Set `Message ID` to `01070410` (or any other random number).
6. Under `Log Publisher` select `perimeterx-publisher`.
7. Click `Finished`.

![alt text](/screenshots/perimeterx-filter_configure.gif)

### Configure PerimeterX iRule

1. Create a new iRule named `px`.
2. Copy the content of `px.tcl` into the `px` iRule and update the following keys under RULE_INIT:

- **APP_ID** - The PerimeterX custom application id in the format of \_PX**\_**\_\_ .
- **AUTH_TOKEN** - The JWT token used for REST API. The Authentication Token is generated in PerimeterX Portal -> **Application** page.
- **COOKIE_SECRET** - The key used by the cookie signing page. The Cookie Key is generated in the PerimeterX Portal -> **Policy** page.

> Note: If you configured a pool for PerimeterX activities report (`px_syslog_pool`) - uncomment the `CLIENT_ACCEPTED` rule (lines 2-4) in the iRule editor.

3. Add the `px` iRule to the Virtual Server you want to protect with PerimeterX.

![alt text](/screenshots/px_irule_assign_vip.gif)

## <a name="firstParty"></a> First-Party Integration

First-Party Mode enables the module to send/receive data to/from the sensor, acting as a reverse-proxy for client requests and sensor activities. To setup First-Party mode on F5 BigIP:

### Configuring Pool: px_client_pool

1. Under `Local Traffic` -> `Pools` -> `Pool List`, create a new pool.
2. Set the pool name to: `px_client_pool`.
3. Set `Health Monitor` to tcp_half_open.
4. Select `new FQDN Node`.
5. Set `Node Name` and `Address` to `client.perimeterx.net`
6. Set the `Service Port` to 443.
7. Set `Auto Populate` to Enabled.
8. Click `Add` & `Finished`.

### Configuring Pool: px_captcha_pool

1. Under `Local Traffic` -> `Pools` -> `Pool List`, create a new pool.
2. Set the pool name to: `px_captcha_pool`.
3. Set `Health Monitor` to tcp_half_open.
4. Select `new FQDN Node`.
5. Set `Node Name` and `Address` to `captcha.px-cdn.net`
6. Set the `Service Port` to 443.
7. Set `Auto Populate` to Enabled.
8. Click `Add` & `Finished`.

### Configuring Pool: px_xhr_pool

1. Under `Local Traffic` -> `Pools` -> `Pool List`, create a new pool.
2. Set the pool name to: `px_xhr_pool`.
3. Set `Health Monitor` to tcp_half_open.
4. Select `new FQDN Node`.
5. Set `Node Name` and `address` to `collector-<APP_ID>.perimeterx.net`

> Note: App id can be retrieved from the PerimeterX portal under Admin->Applications.

6. Set the `Service Port` to 443.
7. Set `Auto Populate` to Enabled.
8. Click `Add` & `Finished`

## <a name="upgrading"></a> Upgrading

Contact [PerimeterX Support](mailto:support@perimeterx.com>) or your assigned PerimeterX Solution Architect for details.

### <a name="basic-usage"></a> Basic Usage Example

```
when HTTP_REQUEST {
    set module_version "F5 BIG-IP 2.5.0"
    set app_id [class lookup "app_id" pxconfig]
    set cookie_secret_key [class lookup "cookie_secret_key" pxconfig]
    set auth_token [class lookup "auth_token" pxconfig]
    set enable_module [class lookup "enable_module" pxconfig]
    set module_mode [class lookup "module_mode" pxconfig]
    set whitelisted_routes_class [class lookup "whitelisted_routes_class" pxconfig]
    set specific_routes_class [class lookup "specific_routes_class" pxconfig]
    set sensitive_routes_class [class lookup "sensitive_routes_class" pxconfig]
    set send_page_activities 1
    set send_block_activities 1
    set excluded_extensions "\.(css|bmp|tif|ttf|docx|woff2|js|pict|tiff|eot|xlsx|csv|eps|woff|xls|jpeg|jpg|doc|ejs|otf|pptx|gif|pdf|swf|svg|ps|ico|pls|midi|svgz|class|png|ppt|mid|webp|jar)$"
    set risk_vs [class lookup "risk_vs" pxconfig]
    set risk_timeout [class lookup "risk_timeout" pxconfig]
    set debug 0
    set ip_header [class lookup "ip_header" pxconfig]
    set sensitive_headers [list "cookie"]
    set custom_logo [class lookup "custom_logo" pxconfig]
    set js_ref [class lookup "js_ref" pxconfig]
    set css_ref [class lookup "css_ref" pxconfig]
    set collector_url "https://collector-${app_id}.perimeterx.net"
    set allowed_domains [list ""]
    set whitelist_ips [list ""]
    set enable_module_header_name [class lookup "enable_module_header_name" pxconfig]
    set bypass_monitor_header [class lookup "bypass_monitor_header" pxconfig]
    set enable_advanced_blocking_response [class lookup "enable_advanced_blocking_response" pxconfig]
    set custom_cookie_header [class lookup "custom_cookie_header" pxconfig]
    set enable_first_party [class lookup "enable_first_party" pxconfig]
    set enable_sensor_injection [class lookup "enable_sensor_injection" pxconfig]
}
```

### <a name="configuration"></a> Configuration Options

Enforcer configuration is stored in "Data Group" named `pxconfig`.

#### Configure Data Groups

1. Edit the `px-datagroup.txt` file and replace all instances of `APP_ID` with PerimeterX's app id.
2. In the same file, replace `AUTH_TOKEN` and `COOKIE_SECRET` with the correct PerimeterX values.
3. `ssh` to the F5 instance and run the content of `px-datagroup.txt` in the shell. This will create the configuration data group for the iRule.

#### Modifying Data Group Items

The following command could be used to modify a single data-group item:
`modify /ltm data-group internal pxconfig records modify { KEY_NAME { data VALUE } }`

Where:

- `KEY_NAME` is the name of the configuration
- `VALUE` is the value

Example (change Enforcer to "blocking" mode):
`modify /ltm data-group internal pxconfig records modify { module_mode { data 2 } }`

#### Configuration required parameters

- `app_id`
- `cookie_key`
- `auth_token`

#### <a name="score-config"></a> Block Score Configuration

Configuring the block score is done in the portal

BIGIP F5 Enforcer uses a binary cookie. The binary cookie does not store the score value on the cookie on the parsed jSON; it is simply being calculated

In order to set a blocking threshold to the binary cookie:

1. Log into the PerimeterX Portal
2. On Admin tab select POLICIES
3. Select Risk Cookie drop-down menu
4. Select "Advanced Mode" and press Continue
5. Unselect v1/v3 if selected and select v2, the binary score should be un-greyed
6. Set a value and apply changes

### <a name="directives"></a> Directives

| Directive                         | Description                                                                                    |             Default              | Value  |
| :-------------------------------- | :--------------------------------------------------------------------------------------------- | :------------------------------: | :----: |
| app_id                            | PX applic ation id                                                                             |               NONE               | String |
| cookie_secret_key                 | Cookie hashing secret (salt)                                                                   |               NONE               | String |
| auth_token                        | JWT used to authenticate with px servers                                                       |               NONE               | String |
| enable_module                     | Sets the module on/off                                                                         |                1                 |  int   |
| module_mode                       | Sets the module working mode, 2 - blocking, 1 - sync monitor, 0 - async monitor                |           2 - blocking           |  int   |
| whitelisted_routes_class          | Name of the class for whitelist routes, please read additional information                     | px\_<APP_ID>\_whitelisted_routes | String |
| whitelisted_query_params_class    | Name of the class for whitelist query parameters                                               | px\_<APP_ID>\_whitelisted_query_params | String |
| specific_routes_class             | Name of the class for specific routes, please read additional information                      |    px_APP_ID_specific_routes     | String |
| sensitive_routes_class            | Name of the class for sensitive routes, please read additional information                     |    px_APP_ID_specific_routes     | String |
| send_page_activities              | Toggles send page requested activity                                                           |                0                 |  int   |
| send_block_activities             | Toggles send block activities                                                                  |                1                 |  int   |
| excluded_extensions               | Flags which extentions the moudle will skip                                                    |           regex String           | String |
| risk_vs                           | Correlates with the virtual server for making risk api calls                                   |      px_backend_APP_ID_vip       | String |
| risk_timeout                      | Sets the timeout for api calls (in milliseconds                                                |               2500               |  int   |
| debug                             | Toggles debug mode on/off, see troubleshooting for more information                            |                0                 |  int   |
| ip_header                         | Custom user header that contains real user ip                                                  |               NONE               | String |
| sensitive_headers                 | List of sensitive headers not to send in risk api calls                                        |            ["cookie"]            |  list  |
| custom_logo                       | Path to url that contains a logo to be displayed on default block page                         |               NONE               | String |
| jf_ref                            | Path to url that contains a custom js file to inject into the default block page               |               NONE               | String |
| css_ref                           | Path to url that contains a custom css file to inject into the default block page              |               NONE               | String |
| allowed_domains                   | A list of domain names on which the enforcer will run on. Run on all if blank                  |               [""]               |  list  |
| enable_module_header_name         | The header name that should be used to enable the module (The header's value should be `True`) |               None               | String |
| whitelist_ips                     | A list of ips/CIDRs to whitelist. If empty all the requests will be processed.                 |               [""]               |  list  |
| bypass_monitor_header             | The header name that can be used to bypass monitor mode on blocking activities.                |               NONE               | String |
| enable_advanced_blocking_response | Toggles the use of advanced blocking response                                                  |                1                 |  int   |
| custom_cookie_header              | A header name which will be used to extract the PerimeterX cookie from.                        |               NONE               | String |
| enable_first_party                | Toggles first party mode on/off.                                                               |                1                 |  int   |
| enable_sensor_injection           | Toggles sensor injection on/off                                                                |                0                 |  int   |

### <a name="additional_information"></a> Additional Information

#### Directives containing _app_id_

Some of the directives on the config may require a specific name that contains the appID of the application taken from the portal.

The name on the configuration must be identical to the name configured on the data group/virtual server/pool.

In case there is a mismatch on the name it may lead to errors on the module

#### Async Activities

Activities such as `page_requested`, `block` or `async_monitor` are sent through Syslog. This allows the module to send these activities to the PerimeterX backend without blocking the request.

#### URI Delimiters

PerimeterX processes URI paths with general- and sub-delimiters according to RFC 3986. General delimiters (e.g., `?`, `#`) are used to separate parts of the URI. Sub-delimiters (e.g., `$`, `&`) are not used to split the URI as they are considered valid characters in the URI path.

### <a name="enable_module_header"></a> Enable Module Header

This configuration enables the module only when the configured header is present and its value is set to `True`.
If `enable_module_header_name` configuration is empty then the `module_enabled` configuration will determine if the module is enabled or not.

Example:

```tcl
when HTTP_REQUEST {
...
    set send_page_activities 1
    set send_block_activities 1
    set enable_module_header_name "X-PX-ENABLE-MODULE"

}
```

### <a name="test_block_flow_on_monitoring_mode"></a> Test Block Flow on Monitoring Mode

Allows you to test an enforcer’s blocking flow while you are still in Monitor Mode.

When the header name is set(eg. `x-px-block`) and the value is set to `1`, when there is a block response (for example from using a User-Agent header with the value of `PhantomJS/1.0`) the Monitor Mode is bypassed and full block mode is applied. If one of the conditions is missing you will stay in Monitor Mode. This is done per request.
To stay in Monitor Mode, set the header value to `0`.

The Header Name is configurable using the `bypass_monitor_header` property.

```tcl
when HTTP_REQUEST {
...
    set bypass_monitor_header "x-px-block"
...
}
```

### <a name="enrich_custom_parameters"></a> Enrich Custom Parameters

The `px_add_custom_parameters` function allows you to add up to 10 custom parameters to be returned to PerimeterX servers on `risk_api` calls. When configured, the function is called before setting the payload on every `risk_api` request to PerimeterX servers. The parameters should be passed in the correct order (ie. if `userid` is set to `custom-param1` in the PerimeterX console, it should always be sent as `x-px-custom-param1`).

To add a custom parameter, add the header key (for exapmle: `x-px-custom-param<number>`) and value to the `custom_parameters` list in the `px_add_custom_parameters` function. The header key will be evaluated before being sent to ensure the right pattern (`x-px-custom-param<x>`) is used.

```tcl
proc px_add_custom_parameters {} {
  # Custom function to add custom parameters to risk_api and async activities.
  set custom_parameters [list]
  # INSERT LOGIC HERE
  lappend custom_parameters "x-px-custom-param1" "UID"
  lappend custom_parameters "x-px-custom-param3" "SessionID"

  # The function must always return the custom_parameters list.
  return $custom_parameters
}
```

### <a name="inject_sensor"></a> JavaScript Sensor Injection

PerimeterX's iRule is able to inject the PerimeterX JavaScript sensor to HTML pages, by setting the `enable_sensor_injection` configuration property to `1`. When the configuration is enabled, the iRule will remove the `Accept-Header` from the incoming request so the origin will **NOT** return the content compressed, even if the client accepts compression. Make sure to enable compression on the LTM level if you use script injection.

### <a name="scoreHeader"></a> Upstream Score Header

When set, this property enables sending a header with the name of `x-px-score` containing the PerimeterX score to the origin.

Possible values:

- 0 - means header is disabled
- 1 - means header is enabled

**Default:** 0

```tcl
enable_score_header: 1
```

### <a name="loginCredentialsExtraction"></a>Login Credentials Extraction

This feature extracts credentials (hashed username and password) from requests and sends them to PerimeterX as additional info in the risk api call. The feature can be toggled on and off, and may be set for a specific path.

**configuration options:**

- ci_login_credentials_extraction_enabled - enable/disable the login extraction feature. Possible values: `1` (enable), `0` (disable).
- ci_login_credentials_extraction_class - name of the class containing the specific login extraction endpoint details.
- ci_credentials_intelligence_version - the version of credentials intelligence to use. Possible values: `multistep_sso` for multi step login process, `v2` for standard mode.
- ci_compromised_credentials_header - name of the header to use for sending the compromised credentials result back to the origin.
- ci_additional_s2s_activity_header_enabled - enable/disable the feature for sending the s2s activity back as a header to the origin. This feature is used for sending additional data on the request back to PerimeterX's server from the origin. Possible values: `1` (enable), `0` (disable).
- ci_send_raw_username_on_additional_s2s_activity - enable/disable sending the raw user name back on the s2s activity header. Possible values: `1` (enable), `0` (disable).
- ci_login_successful_reporting_method - reporting method of a successful login. Possible values: `header` for header name/value configuration, `status` for a specific status code indicating a successful login.
  - ci_login_successful_header_name - specify a header name to extract a successful login indicator. Will only be used if `ci_login_successful_reporting_method` is set to `header`.
  - ci_login_successful_header_value - specify the header value that indicate a successful login indicator. Will only be used if `ci_login_successful_reporting_method` is set to `header`.
  - ci_login_successful_status - specify a status code indicating a successful login. Will only be used if `ci_login_successful_reporting_method` is set to `status`.

Configure the extraction in the class configured in `ci_login_credentials_extraction_class`:

- path: The path for the login request, for example: `/path`.
- method: The HTTP method used for submiting the login request, for example :`POST`.
- sent_through: What is the type of payload the credentials are passing by. Possible values: `body`, `header` or `query-string`.
- user_field: The user name field name, for example: `username`.
- pass_field: The password field name, for example: `password`.

### <a name="userIdentifiers"></a>User Identifiers

Enable the extraction of JWT fields from requests and add them to the risk, page requested and block activities.

**configuration options:**

- jwt_cookie_name: The cookie name that should contain the JWT token.
- jwt_cookie_user_id_field_name: The field name in the JWT object, extracted from the JWT cookie, that contains the user ID to be extracted.
- jwt_cookie_additional_field_names: The field names in the JWT object, extracted from the JWT cookie, that should be extracted in addition to the user ID.
- jwt_header_name: The header name that should contain the JWT token.
- jwt_header_user_id_field_name: The field name in the JWT object, extracted from the JWT header, that contains the user ID to be extracted.
- jwt_header_additional_field_names: The field names in the JWT object, extracted from the JWT header, that should be extracted in addition to the user ID.

## <a name="configurational_classes"></a> Configurational Classes

### Whitelisted Routes

1. On `Local Traffic` -> `iRules` -> `Data Group List` create a new Data Group.
2. Set the `Name` to `px_<APP_ID>_whitelisted_routes`. Make sure to replace `<APP_ID>` with your PerimeterX app id.
3. Set `Type` to `String`.
4. For each route you wish to whitelist, set the `String` property of `String Record` to the prefix of the route and click `Add`.
5. Click `Finished`.

![alt text](/screenshots/whitelisted_routes.png)

### Whitelisted URI query paremeters

1. On `Local Traffic` -> `iRules` -> `Data Group List` create a new Data Group.
2. Set the `Name` to `px_<APP_ID>_whitelisted_query_params`. Make sure to replace `<APP_ID>` with your PerimeterX app id.
3. Set `Type` to `String`.
4. For each URI query parameter you wish to whitelist, set the `String` property to the query key and `Value` property to the query **regex** value and click `Add`.
5. Click `Finished`.

![alt text](/screenshots/whitelisted_query_params.png)

### Specific Routes

1. On `Local Traffic` -> `iRules` -> `Data Group List` create a new Data Group.
2. Set the `Name` to `px_<APP_ID>_specific_routes`. Make sure to replace `<APP_ID>` with your PerimeterX app id.
3. Set `Type` to `String`.
4. For each route you wish to whitelist, set the `String` property of `String Record` to the prefix of the route and click `Add`.
5. Click `Finished`.

![alt text](/screenshots/specific_routes.png)

### Sensitive Routes

1. On `Local Traffic` -> `iRules` -> `Data Group List` create a new Data Group.
2. Set the `Name` to `px_<APP_ID>_sensitive_routes`. Make sure to replace `<APP_ID>` with your PerimeterX app id.
3. Set `Type` to `String`.
4. For each route you wish to whitelist, set the `String` property of `String Record` to the prefix of the route and click `Add`.
5. Click `Finished`.

![alt text](/screenshots/sensitive_routes.png)

## <a name="advanced_blocking_response"></a> Advanced Blocking Response

In special cases, (such as XHR post requests) a full Captcha page render might not be an option. In such cases, using the Advanced Blocking Response returns a JSON object continaing all the information needed to render your own Captcha challenge implementation, be it a popup modal, a section on the page, etc. The Advanced Blocking Response occurs when a request contains the _Accept_ header with the value of `application/json`. A sample JSON response appears as follows:

```javascript
{
    "appId": String,
    "jsClientSrc": String,
    "firstPartyEnabled": Boolean,
    "vid": String,
    "uuid": String,
    "hostUrl": String,
    "blockScript": String
}
```

Once you have the JSON response object, you can pass it to your implementation (with query strings or any other solution) and render the Captcha challenge.

In addition, you can add the `_pxOnCaptchaSuccess` callback function on the window object of your Captcha page to react according to the Captcha status. For example when using a modal, you can use this callback to close the modal once the Captcha is successfullt solved. <br/> An example of using the `_pxOnCaptchaSuccess` callback is as follows:

```javascript
window._pxOnCaptchaSuccess = function (isValid) {
    if (isValid) {
        alert("yay");
    } else {
        alert("nay");
    }
};
```

For Advanced Blocking Resposne sample please refer to the [abr-samples](https://github.com/PerimeterX/perimeterx-abr-samples) Github repository.

### <a name="troubleshooting"></a> Troubleshooting

To troubleshoot the iRule added to the F5 BIG-IP, ssh into the BIG-IP machine, type `bash` in the command prompt and then check the ltm logs located in /var/log/ltm

```bash
tail -f /var/log/ltm
```
