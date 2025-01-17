# requests-ip-rotator
A Python library to utilize AWS API Gateway's large IP pool as a proxy to generate pseudo-infinite IPs for web scraping and brute forcing.

This library will allow the user to bypass IP-based rate-limits for sites and services.

---
## Installation
This package is on pypi so you can install via any of the following:  
* `pip3 install requests-ip-rotator`
* `python3 -m pip install requests-ip-rotator`


&nbsp;
## Simple Usage
```py
import requests
from requests_ip_rotator import ApiGateway

# Create gateway object and initialise in AWS
gateway = ApiGateway("https://site.com")
gateway.start()

# Assign gateway to session
session = requests.Session()
session.mount("https://site.com", gateway)

# Send request (IP will be randomised)
response = session.get("https://site.com/index.html")
print(response.status_code)

# Delete gateways
gateway.shutdown()
```

Please remember that if gateways are not shutdown via the `shutdown()` method, you may be charged in future.

&nbsp;
## Costs
API Gateway is free for the first million requests per region, which means that for most use cases this should be completely free.  
At the time of writing, AWS charges ~$3 per million requests after the free tier has been exceeded.  
&nbsp;

## Documentation
### AWS Authentication
It is recommended to setup authentication via environment variables. With [awscli](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html), you can run `aws configure` to do this, or alternatively, you can simply set the `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` variables yourself.

You can find your access key ID and secret by following the [official AWS tutorial](https://docs.aws.amazon.com/powershell/latest/userguide/pstools-appendix-sign-up.html).

&nbsp;
### Creating ApiGateway object
The ApiGateway class can be created with the following optional parameters:  
| Name              | Description                                          | Required    | Default
| -----------       | -----------                                          | ----------- | -----------
| site              | The site (without path) requests will be sent to.    | True        |
| regions           | An array of AWS regions to setup gateways in.        | False       | ip_rotator.DEFAULT_REGIONS
| access_key_id     | AWS Access Key ID (will override env variables).     | False       | *Relies on env variables.*
| access_key_secret | AWS Access Key Secret (will override env variables). | False       | *Relies on env variables.*
```python
from ip_rotator import ApiGateway, EXTRA_REGIONS, ALL_REGIONS

# Gateway to outbound HTTP IP and port for only two regions
gateway_1 = ApiGateway("http://1.1.1.1:8080", regions=["eu-west-1", "eu-west-2"])

# Gateway to HTTPS google for the extra regions pack, with specified access key pair
gateway_2 = ApiGateway("https://www.google.com", regions=EXTRA_REGIONS, access_key_id="ID", access_key_secret="SECRET")

```

&nbsp;
### Starting API gateway
An ApiGateway object must then be started using the `start` method.  
**By default, if an ApiGateway already exists for the site, it will use the existing endpoint instead of creating a new one.**  
This does not require any parameters, but accepts the following:
| Name              | Description                                                   | Required    |
| -----------       | -----------                                                   | ----------- |
| force             | Create a new set of endpoints, even if some already exist.    | False       |
| endpoints         | Array of pre-existing endpoints (i.e. from previous session). | False       |
```python
# Starts new ApiGateway instances for site, or locates existing endpoints if they already exist.
gateway_1.start()

# Starts new ApiGateway instances even if some already exist.
gateway_2.start(force=True)
```

&nbsp;
### Sending requests
Requests are sent by attaching the ApiGateway object to a requests Session object.  
The site given in `mount` must match the site passed in the `ApiGateway` constructor.  

If you pass in a `X-My-X-Forwarded-For` header, this will get sent as `X-Forwarded-For` in the outbound request.
```python
import requests

# Posts a request to the site created in gateway_1. Will be sent from a random IP.
session_1 = requests.Session()
session_1.mount("http://1.1.1.1:8080", gateway_1)
session_1.post("http://1.1.1.1:8080/update.php", headers={"Hello": "World"})

# Send 127.0.0.1 as X-Forwarded-For header in outbound request.
session_1.post("http://1.1.1.1:8080/update.php", headers={"X-My-X-Forwarded-For", "127.0.0.1"})

# Execute Google search query from random IP
session_2 = requests.Session()
session_2.mount("https://www.google.com", gateway_2)
session_2.get("https://www.google.com/search?q=test")
```

&nbsp;
### Closing ApiGateway Resources
It's important to shutdown the ApiGateway resources once you have finished with them, to prevent dangling public endpoints that can cause excess charges to your account.  
This is done through the `shutdown` method of the ApiGateway object. It will close all resources for the regions specified in the ApiGateway object constructor.

```python
# This will shutdown all gateway proxies for "http://1.1.1.1:8080" in "eu-west-1" & "eu-west-2"
gateway_1.shutdown()

# This will shutdown all gatewy proxies for "https://www.google.com" for all regions in ip_rotator.EXTRA_REGIONS
gateway_2.shutdown()
```

## Credit
The core gateway creation and organisation code was adapter from RhinoSecurityLabs' [IPRotate Burp Extension](https://github.com/RhinoSecurityLabs/IPRotate_Burp_Extension/).  
The X-My-X-Forwarded-For header forwarding concept was originally conceptualised by [ustayready](https://twitter.com/ustayready) in his [fireprox](https://github.com/ustayready/fireprox) proxy.