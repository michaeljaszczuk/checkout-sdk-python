# Checkout.com Python SDK

[![Build Status](https://travis-ci.org/checkout/checkout-sdk-python.svg?branch=master)](https://travis-ci.org/checkout/checkout-sdk-python)
[![codecov](https://codecov.io/gh/checkout/checkout-sdk-python/branch/master/graph/badge.svg)](https://codecov.io/gh/checkout/checkout-sdk-python)

## Installation

    pip install --upgrade checkout-sdk

From source:

    python setup.py install

### Requirements

* Python 3.4+

## Usage

### Using environment variables

``` python
os.environ['CKO_SECRET_KEY'] = '<your secret key>'
os.environ['CKO_SANDBOX'] = 'True|true|1'               # else is False (Production)
os.environ['CKO_LOGGING'] = 'debug|DEBUG|info|INFO'

# ...

import checkout_sdk as sdk

api = sdk.get_api()
```

### Using initialisation values

``` python
import checkout_sdk as sdk

api = sdk.get_api(secret_key='<your secret key>')       # default sandbox = True
```

### Setting defaults

``` python
sdk.default_currency = sdk.Currency.EUR
sdk.default_auto_capture = True
sdk.default_auto_capture_delay = 0                      # valid: 0 - 168 (hours)
sdk.default_payment_type = sdk.PaymentType.Regular
```

### Payment Request

#### Full Card

``` python
try:
    payment = api.payments.request(
        card={
            'number': '4242424242424242',
            'expiryMonth': 6,
            'expiry_year': 2025,                        # snake_case is auto converted
            'cvv': '100'
        },
        value=100,                                      # cents
        currency=sdk.Currency.USD,                      # or 'usd'
        customer='customer@email.com'
    )
    print(payment.id)
    print(payment.card.id)
    print(payment.customer.id)
    print(payment.http_response.body)                   # JSON body
except sdk.errors.CheckoutSdkError as e:
    print('{0.http_status} {0.error_code} {0.elapsed} {0.event_id} // {0.message}'.format(e))
```

#### Card Id

``` python
try:
    payment = api.payments.request(
        card='card_713A3978-AFB2-4D30-BF9A-BA55714DC309',
        value=100,                                      # cents
        currency=sdk.Currency.USD,                      # or 'usd'
        customer='customer@email.com'
    )
    if payment.approved:
        # ...
except sdk.errors.CheckoutSdkError as e:
    print('{0.http_status} {0.error_code} {0.elapsed} {0.event_id} // {0.message}'.format(e))
```

### Payment History

#### Get History

```python
try:
    history = api.payments.history(payment_id)          # charge_00000000000000000000
    for charge in history.charges:
        print(charge.id)
        print(charge.created)
        print(charge.response_code)
except sdk.errors.CheckoutSdkError as e:
    print('{0.http_status} {0.error_code} {0.elapsed} {0.event_id} // {0.message}'.format(e))
```

See [payment_history.py](https://github.com/checkout/checkout-sdk-python/blob/master/checkout_sdk/payments/payment_history.py) and [charge.py](https://github.com/checkout/checkout-sdk-python/blob/master/checkout_sdk/common/charge.py) for more info.

### 3DS Support

#### Full Card With 3DS Support

``` python
try:
    payment = api.payments.request(
        card={
            'number': '4242424242424242',
            'expiryMonth': 6,
            'expiryYear': 2025,
            'cvv': '100'
        },
        value=100,
        currency=sdk.Currency.USD,
        customer='customer@email.com',
        charge_mode=sdk.ChargeMode.ThreeDS
    )
    print(payment.requires_redirect)                    # True
    print(payment.id)                                   # Payment Token
    print(payment.redirect_url)                         # ACS Url
    print(payment.http_response.body)                   # JSON body
except sdk.errors.CheckoutSdkError as e:
    print('{0.http_status} {0.error_code} {0.elapsed} {0.event_id} // {0.message}'.format(e))
```

> **Important**: If you use the Checkout.com Risk Engine to upgrade to a 3DS flow (from N3D) depending on criteria, you must always check for `payment.requires_redirect` first.

#### Full Card With 3DS Support + N3D Downgrade Option

``` python
try:
    payment = api.payments.request(
        card={
            'number': '4242424242424242',
            'expiryMonth': 6,
            'expiryYear': 2025,
            'cvv': '100'
        },
        value=5000,
        currency=sdk.Currency.USD,
        customer='customer@email.com',
        charge_mode=sdk.ChargeMode.ThreeDS,
        attempt_n3d=True
    )
    print(payment.downgraded)                           # True
    print(payment.id)                                   # Payment Token
    print(payment.redirect_url)                         # Success/Confirmation Url
except sdk.errors.CheckoutSdkError as e:
    print('{0.http_status} {0.error_code} {0.elapsed} {0.event_id} // {0.message}'.format(e))
```

> **Important**: Value needs to be set to `5000` to simulate a `20153` response code on the Sandbox environment, which will then attempt an N3D charge.

### Tokens

#### Payment Token Request

``` python
try:
    token = api.tokens.request_payment_token(
        value=100,                                      # cents
        currency=sdk.Currency.USD,                      # or 'usd'
        track_id='001',
        success_url='http://success.com',
        fail_url='http://retry.com'
    )
    print(token.id)
    print(token.http_response.body)                     # JSON body
except sdk.errors.CheckoutSdkError as e:
    print('{0.http_status} {0.error_code} {0.elapsed} {0.event_id} // {0.message}'.format(e))
```

> **Important**: The SDK only support payment token creation at present and intended for Alternative Payment Methods via Checkout.js.

### Exception handling

``` python
class CheckoutSdkError(Exception):                      # catch all
class AuthenticationError(CheckoutSdkError):            # 401
class BadRequestError(CheckoutSdkError):                # 400
class ResourceNotFoundError(CheckoutSdkError):          # 404
class Timeout(CheckoutSdkError):
class TooManyRequestsError(CheckoutSdkError):           # 422
class ApiError(CheckoutSdkError):                       # 500 / fallback
```

> The SDK will not do any offline validation of card data, IDs, etc. Provided the values and types are correct, all business validations are handled at API level. On that note, expect `ValueError` and `TypeError` for incorrect usage.

#### Handling API Validation Exceptions

API Response

``` json
{
    "eventId": "00000000-0000-0000-0000-000000000000",
    "errorCode": "70000",
    "message": "Validation error",
    "errorMessageCodes": [
        "70034",
        "70013"
    ],
    "errors": [
        "Invalid card id",
        "Invalid customer id"
    ]
}
```

Exception Handling

``` python
except sdk.errors.BadRequestError as e:
    if e.validation_error:                              # error_code == 70000
        print(e.errors)                                 # dictionary { msg_code: msg }
        for msg_code in e.errors:
            print(msg_code)                             # e.g. 70034
            print(e.errors[msg_code])                   # e.g. Invalid card id
```

### Logging

```python
os.environ['CKO_LOGGING'] = 'debug|DEBUG|info|INFO'
```

or ...

```python
import logging
logging.getLogger('cko').setLevel(logging.DEBUG)
```

### Test Suite

The tests currently need a Sandbox account. This will eventually be replaced by the incoming Checkout.com Mock API.

```
export CKO_SECRET_KEY="<your secret key>"
export CKO_LOGGING="info|debug"
python setup.py test
```
