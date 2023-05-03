# Authentication

Please use basic authentication. That is, each request should contain a header field in the form of Authorization: Basic <credentials>, where credentials is the base64 encoding of username and password joined by a single colon :. Generate via the following python code:
```
import base64
username = 'your_username'
password = 'your_password'
key = f'{username}:{password}'
base64.b64encode(key.encode('utf-8')).decode('utf-8')
```

Please also submit  to Sendwave the IP address from the VM which will be making the requests so we can whitelist it. IP-whitelisting is not required for the sandbox.


# Lookup Transfer: GET /transfers/<confirmation_code>

## Endpoint Details

* Query parameters:
    * *confirmation_code*: (string) code sent to recipient and sender. This is the code presented to the teller by the recipient (a string of variable length up to 20 characters, potentially including uppercase alphanumeric characters and -)
* Response Attributes:
    * *confirmation_code*: (string) code sent to recipient and sender. This is presented code presented to the teller by the recipient (a string of variable length up to 20 characters, potentially including uppercase alphanumeric characters and -)
    * *recipient_name*: (string) Recipients name as the user inputted.
        * Note: If this needs to be updated the Sender can call into Sendwave Support and change the name of the recipient.
    * *recipient_mobile*: (string) Mobile number which Sendwave texted the confirmation_code to (e.g. "+234xxxxxxxxxx")
    * *recipient_country*: (string) ISO Alpha-2 country code that the recipient hails from.
      * Possible countries: ng
    * *sender_name*: (string) Name of the sender
    * *sender_country*: (string) ISO Alpha-2 country code that the sender is from.
      * Possible countries: us, gb, fr, es, it, ir, ie, ca
    * *sender_state*: (string) Two letter state code (if applicable)
      * Possible entries: 2-letter U.S. state code.
    * *send_amount*: (float)
    * *send_currency*: (str) Currency iso_3
        * Possible currencies: USD, GBP, EUR, CAD
    * *receive_amount*: (float)
    * *receive_currency*: (str) Currency iso_3
        * Possible currencies: USD
    * *status*:
        * NOT_PAID: These are transactions Senders have initiated through Sendwave for cash pickup at Access bank. Sendwave is awaiting the recipient to cash out at a bank and an Access Bank teller to complete the transaction
        * LOCKED: These transactions are in the process of going through teller confirmation with the beneficiary (recipient).
        * PAID: These are transactions which have already been paid out
            * Note: Access Bank should not pay these out since they have already been paid out
        * CANCELLED: These are transactions that Senders have initiated through Sendwave for cash pickup at Access bank, but have been cancelled by either the sender or the transfer is past 7 days.
            * Note: Sendwave automatically cancels transfers initiated over 7 days ago that have not been picked up by the recipient.
    * *status_description*: (str) Description of error
        * ‘Cancelled by Sender’
        * ‘Cancelled by system: Over 7 days’

## Sample Requests

### Not Found
```
curl --request GET 'https://app.sendwave.com/transfers/ZZZZZZ' \
    --header 'Authorization: Basic <token>' \
    --w "%{http_code}"

404%
```
### PAID
```
curl --request GET 'https://app.sendwave.com/transfers/AAAAAA' \
    --header 'Authorization: Basic <token>' \
    --w "%{http_code}"

{"confirmation_code": "AAAAAA", "recipient_name": "Maxwell Obi",
"recipient_mobile": "+2347031247953", "send_amount": 95.95,
"send_currency": "USD", "receive_amount": 95.95, "receive_currency": "USD",
"status": "PAID", "status_description": ""}200%
```
### NOT PAID
```
curl --request GET 'https://app.sendwave.com/transfers/BBBBBB' \
    --header 'Authorization: Basic <token>' \
    --w "%{http_code}"

{"confirmation_code": "BBBBBB", "recipient_name": "Maxwell Obi",
"recipient_mobile": "+2347031247953", "send_amount": 95.95,
"send_currency": "USD", "receive_amount": 95.95,
"receive_currency": "USD", "status": "NOT_PAID",
"status_description": ""}200%
```

### CANCELLED
```
curl --request GET 'https://app.sendwave.com/transfers/CCCCCC' \
    --header 'Authorization: Basic <token>' \
    --w "%{http_code}"

{"confirmation_code": "CCCCCC", "recipient_name": "Maxwell Obi",
"recipient_mobile": "+2347031247953", "send_amount": 95.95,
"send_currency": "USD", "receive_amount": 95.95, "receive_currency": "USD",
"status": "CANCELLED",
"status_description": "Cancelled by system: Over 7 days"}200%
```
### LOCKED
```
curl --request POST 'https://app.sendwave.com/transfers/DDDDDD' \
    --header 'Authorization: Basic <token>' \
    --w "%{http_code}"

{"confirmation_code": "DDDDDD", "recipient_name": "Maxwell Obi",
"status": "LOCKED", "status_description": "Transfer already locked for distribution"}200%
```


# Lock Transfer PUT /transfers/<confirmation_code>/lock
This endpoint will lock the transaction for 15 minutes and prevent other banks/tellers from processing the transaction for that duration. This can be called immediately after the GET lookup above.

## Endpoint Details:

* Query parameters:
    * *confirmation_code*: (string) code sent to recipient and sender. This is presented code presented to the teller at Access Bank  (a string of variable length up to 20 characters, potentially including uppercase alphanumeric characters and -)
* Request data
    * *branch_code *(REQUIRED)*: *Unique identifier for the bank branch
    * *employee_id *(REQUIRED): Unique identifier for the bank employee performing the payout
* Response Attributes:
    * *confirmation_code*: (string)  code sent to recipient and sender. This is the code presented to the teller at Access Bank by the recipient  (a string of variable length up to 20 characters, potentially including uppercase alphanumeric characters and -)
    * *status: *
        * LOCKED
        * CANNOT_LOCK
            * e.g.:
                * Already locked
                * Transfer has status: PAID
                * Transfer has status: CANCELLED
        * FAILED
            *  OTP failed to send
    * *status_description:* (str) Description of error
        * Already locked
        * Transfer has status: PAID
        * Transfer has status: CANCELLED
        *  OTP failed to send

## Sample Requests:

### Lock
```
curl --location --request PUT 'https://app.sendwave.com/transfers/AAAAAA/lock' --header 'Content-Type: application/json' --header 'Authorization: Basic <token>' —data-raw '{
"branch_code": "XYZ456",
"employee_id": "123456"
}‘
{"confirmation_code": "AAAAAA", "status": "LOCKED"}
200
```
### Already Locked
```
curl --location --request PUT 'https://app.sendwave.com/transfers/BBBBBB/lock' --header 'Content-Type: application/json' --header 'Authorization: Basic <token>' —data-raw '{
"branch_code": "XYZ456",
"employee_id": "123456"
}‘
{"confirmation_code": "BBBBBB", "status": "UNABLE_TO_LOCK", "description": "Transaction Already Locked"}
409
```
### Already Paid
```
curl --location --request PUT 'https://app.sendwave.com/transfers/CCCCCC/lock' --header 'Content-Type: application/json' --header 'Authorization: Basic <token>' —data-raw '{
"branch_code": "XYZ456",
"employee_id": "123456"
}‘
{"confirmation_code": "CCCCCC", "status": "UNABLE_TO_LOCK", "description": "Transaction Already Paid"}
409
```
### Cancelled
```
curl --location --request PUT 'https://app.sendwave.com/transfers/DDDDDD/lock' --header 'Content-Type: application/json' --header 'Authorization: Basic <token>' —data-raw '{
"branch_code": "XYZ456",
"employee_id": "123456"
}‘
{"confirmation_code": "DDDDDD", "status": "UNABLE_TO_LOCK", "description": "Transaction Cancelled"}
409
```
### Unable to send otp code
```
curl --location --request PUT 'https://app.sendwave.com/transfers/EEEEEE/lock' --header 'Content-Type: application/json' --header 'Authorization: Basic <token>' —data-raw '{
"branch_code": "XYZ456",
"employee_id": "123456"
}‘
{"confirmation_code": "EEEEEE", "status": "FAILURE", "description": "Unable to send recipient OTP Code"}
500
```
### NOT FOUND:
```
curl --request PUT  'https://app.sendwave.com/transfers/ZZZZZZ/lock'\
    --header 'Authorization: Basic <token>' \
    -d \
    "{
        \"branch_code\": ""\"XYZ456\",
        \"employee_id\": \"123456\",
     }" \
    --w "%{http_code}"

404%
```



# Mark Transfer Complete PUT /transfers/<confirmation_code>

## Endpoint Details

* Query parameters:
    * *confirmation_code*: (string) code sent to recipient and sender. This is presented code presented to the teller at Access Bank  (a string of variable length up to 20 characters, potentially including uppercase alphanumeric characters and -)
* Request data
    * *track_id *(REQUIRED):* *Transaction id on the bank side
    * *branch_code *(REQUIRED)*: *Unique identifier for the bank branch
    * *employee_id *(REQUIRED): Unique identifier for the bank employee performing the payout
    * *otp *(REQUIRED): User supplied OTP. Must match the one that they were sent.
* Response Attributes
    * *confirmation_code*: (string)  code sent to recipient and sender. This is the code presented to the teller at Access Bank by the recipient  (a string of variable length up to 20 characters, potentially including uppercase alphanumeric characters and -)
    * *status*: (str)
        * PAY_OUT: The teller should pay out the funds. The transfer has been validated and marked successful in Sendwave’s system.
        * ALREADY_PAID: Do not pay out. The transfer has not been validated and has already been completed. This status may include an additional field, `status_description="SAME_BANK_BRANCH_EMPLOYEE"`, indicating that the current "Mark transfer complete" request shares the same bank, branch, and employee ID as a previous request. This is designed to resolve any failed requests due to network or system issues by informing the user that if (and only if) their previous request was unsuccessful, the teller is still able to pay out the funds.
        * CANCELLED: Do not pay out. The transfer has been cancelled by the sender or Sendwave.
        * INVALID_OTP: Do not pay out. An invalid OTP was submitted. Teller may re-submit.
    * *status_description*: (str) Description of error
        * e.g.:
            * ‘Cancelled by Sender’
            * ‘Cancelled by system: Over 7 days’

## Sample Requests

### BAD REQUEST
```
curl --request PUT 'https://app.sendwave.com/transfers/AAAAAA' \
    --header 'Authorization: Basic <token>' \
    --w "%{http_code}"

Missing request data400%
```
### NOT FOUND
```
curl --request PUT 'https://app.sendwave.com/transfers/ZZZZZZ' \
  --header 'Authorization: Basic <token>' \
  --header 'Content-Type: application/json' \
  -d \
    "{
        \"track_id\": \"abc123\",
        \"branch_code\": ""\"XYZ456\",
        \"employee_id\": \"123456\",
        \"otp\": \"OTP\"}" \
  --w "%{http_code}"

404%
```
### PAY_OUT
```
curl --request PUT 'https://app.sendwave.com/transfers/BBBBBB' \
  --header 'Authorization: Basic <token>' \
  --header 'Content-Type: application/json' \
  -d \
    "{
        \"track_id\": \"abc123\",
        \"branch_code\": ""\"XYZ456\",
        \"employee_id\": \"123456\",
        \"otp\": \"ABC123\"}" \
  --w "%{http_code}"

{"confirmation_code": "BBBBBB", "status": "PAY_OUT",
"status_description": ""}200%
```

### ALREADY_PAID
```
curl --request PUT 'https://app.sendwave.com/transfers/AAAAAA' \
  --header 'Authorization: Basic <token>' \
  --header 'Content-Type: application/json' \
  -d \
    "{
        \"track_id\": \"abc123\",
        \"branch_code\": ""\"XYZ456\",
        \"employee_id\": \"123456\",
        \"otp\": \"OTP\"}" \
  --w "%{http_code}"

{"confirmation_code": "AAAAAA", "status": "ALREADY_PAID",
"status_description": ""}200%
```

### CANCELLED
```
curl --request PUT 'https://app.sendwave.com/transfers/CCCCCC' \
  --header 'Authorization: Basic <token>' \
  --header 'Content-Type: application/json' \
  -d \
    "{
        \"track_id\": \"abc123\",
        \"branch_code\": ""\"XYZ456\",
        \"employee_id\": \"123456\",
        \"otp\": \"OTP\"}" \
  --w "%{http_code}"

{"confirmation_code": "CCCCCC", "status": "CANCELLED",
"status_description": "Cancelled by system: Over 7 days"}200%
```
### INVALID_OTP
```
curl --request PUT 'https://app.sendwave.com/transfers/BBBBBB' \
  --header 'Authorization: Basic <token>' \
  --header 'Content-Type: application/json' \
  -d \
    "{
        \"track_id\": \"abc123\",
        \"branch_code\": ""\"XYZ456\",
        \"employee_id\": \"123456\",
        \"otp\": \"INVALID\"}" \
  --w "%{http_code}"

{"confirmation_code": "BBBBBB", "status": "INVALID_OTP",
"status_description": ""}200%
```

# Resend OTP POST /transfers/<confirmation_code>/locks/notifications

## Endpoint Details

* Query parameters:
    * *confirmation_code*: (string) code sent to recipient and sender. This is the code presented to the teller at the bank  (a string of variable length up to 20 characters, potentially including uppercase alphanumeric characters and -)

* Response Attributes
    * *confirmation_code*: (string)  code sent to recipient and sender. This is the code presented to the teller at the bank by the recipient  (a string of variable length up to 20 characters, potentially including uppercase alphanumeric characters and -)
    * *status*: (str)
        * OTP_NOTIFICATION_SENT: The OTP notification was sent (but not necessarily delivered) to the recipient.
        * OTP_NOTIFICATION_NOT_SENT: The system did not attempt to send an OTP (e.g. because the given confirmation code did not correspond to a locked transaction)
        * OTP_NOTIFICATION_FAILED: The system attempted to send the OTP but the attempt failed.
    * *status_description*: (str) Description of error
        * e.g.:
            * ‘Sent OTP’
            * ‘Transaction not locked’
            * ‘Unable to send recipient OTP Code’
            * ‘No transaction found for confirmation code’

## Sample Requests

### OTP_NOTIFICATION_SENT
```
curl --request POST 'https://app.sendwave.com/transfers/AAAAAA/locks/notifications' \
--header 'Authorization: Basic <token>' \
--header 'Content-Type: application/json' \
--w "%{http_code}"

{"confirmation_code": "AAAAAA", "status": "OTP_NOTIFICATION_SENT", "status_description": "Sent OTP"}201%
```

### OTP_NOTIFICATION_NOT_SENT: Transaction Not Found
```
curl --request POST 'https://app.sendwave.com/transfers/DDDDDD/locks/notifications' \
--header 'Authorization: Basic <token>' \
--header 'Content-Type: application/json' \
--w "%{http_code}"
{"confirmation_code": "DDDDDD", "status": "OTP_NOTIFICATION_NOT_SENT", "status_description": "No transaction found for confirmation code"}404%
```

### OTP_NOTIFICATION_NOT_SENT: Transaction Not Locked
```
curl --request POST 'https://app.sendwave.com/transfers/BBBBBB/locks/notifications' \
--header 'Authorization: Basic <token>' \
--header 'Content-Type: application/json' \
--w "%{http_code}"
{"confirmation_code": "BBBBBB", "status": "OTP_NOTIFICATION_NOT_SENT", "status_description": "Transaction not locked"}422%
```

### OTP_NOTIFICATION_FAILED
```
curl --request POST 'https://app.sendwave.com/transfers/CCCCCC/locks/notifications' \
--header 'Authorization: Basic <token>' \
--header 'Content-Type: application/json' \
--w "%{http_code}"
{"confirmation_code": "CCCCCC", "status": "OTP_NOTIFICATION_FAILED", "status_description": "Unable to send recipient OTP Code"}500%
```
