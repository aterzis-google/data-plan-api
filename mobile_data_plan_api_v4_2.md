## Public Data Plan API v4.2

### gtaf-eng@google.com

### May 30th, 2017

Table of Contents

"[TOC]

## 1. Motivation

The goal of this document is to describe the public interfaces that Google will
use to identify the users' mobile data plans, collect information about these
plans, and purchase data plans.

## 2. Changes

The v4.2 API introduces the following changes:

* Addition of application id to DPA requests.
* Addition of error cause in error responses from the DPA.
* Addition of an Eligibility API (see Section [5.7](#5-7-eligibility)).

## 3. Terminology

This document uses the following terms:

*   **CPID.** Carrier Plan Identifier. An opaque string that has a many-to-one
    correspondence with a user's MSISDN.
*   **DPA.** Data Plan Agent. An operator function that returns information
    about users' data plans and account details. The DPA can optionally generate
    CPIDs and make changes to users' data plans (e.g., purchase new plans).
*   **GTAF.** Google Traffic Application Function. A Google function that
    interacts with DPA on behalf of all Google apps.

### 3.1 Requirements Language 

The keywords "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).

## 4. Data Plan Terminology 

The Data Plan API MUST be able to represent data plans defined in terms of
different URLs (e.g., all traffic to *.acmefake.com[^1] is charged at a
different rate) or specific user *journeys* within a single application. We call
the later *sub-app *data plans. An example of a sub-app data plan would be to
offer free (i.e., zero rate) YouTube browsing, while watching videos within the
YouTube application deducts data from the subscriber's data balance. The YouTube
app MUST then be able to learn this information when querying for data plan
information.

Next, we introduce some terms related to data plans. Figure 1 provides examples
of data plans that are representative of the concepts that we want to capture.

*   **Data Plan:** the top-level mobile service package that a subscriber
    purchases. It can be as simple as "10 GB mobile data for 30 days" or it can
    be defined as a collection of components, also known as modules. A data plan
    has:
    *   **Data Plan Name**, such as "ACME Red".
    *   **Data Plan Identifier** used to refer to the plan, for example during
        purchases.
    *   **Expiration time**, when the data plan expires.
    *   **Cost**, price of the data plan.
    *   **Status:** a data plan can be *purchased,* but not activated, or it can
        be *active.*
*   **Plan Module:** a component of a data plan. Specifically a plan module has:

    *   **Plan Module Traffic Category (PMTC):** a description of the data
        traffic that a module applies to. The PMTC can be as general as *all
        Internet traffic *or as specific as traffic generated/consumed by one or
        more applications, websites, or even user journeys within a single
        application. Examples of the later cases are "unlimited music", "100 MB
        Video Data Pack (VDP)", "unlimited gaming data" and "unlimited YouTube
        browsing". To facilitate the definition of PMTCs we have defined the
        following PMTCs : `GENERIC, VIDEO, VIDEO_BROWSING, VIDEO_OFFLINE`[^1]`,
        MUSIC, GAMING, SOCIAL, MESSAGING` and `PMTC_UNSPECIFIED.`

    *   **Data volume or time limit:** once activated, the plan module expires
        when either the data volume or time limit (in the case of time-based
        plans, e.g., 600 minutes of Internet access during the next 7 days) is
        exceeded. In Figure 1 below, a subscriber can buy a plan module, as part
        of "ACME Blue", that provides 1 GB of general user traffic that have to
        be used within a week of activation before they expire.

    *   **Plan Module Priority:** the traffic that an application
        generates/consumes can be mapped to multiple data plan modules. For
        example, consider a user who has purchased a Video Data Plan (VDP) in
        addition to a general purpose data plan. The intention then is that
        traffic from the YouTube app should be charged against the VDP, as long
        as that plan is still valid. Once the VDP expires, YT traffic will be
        charged against the general purpose data plan. In order to capture this
        behavior, each data plan module should have a priority. Then, an
        application can use the traffic categories for each of the data plan
        modules that a user has purchased and their relative priority to
        determine how it will be charged.

![Data Plan](https://github.com/aterzis-google/data-plan-api/blob/master/Data%20Plan%20API%20Sample%20Plan.png)

**Figure 1. Sample data plans.**

## 5. API Description 

At a high level, the proposed API comprises six parts:

1.  Mechanism to establish and update the identify of a user's mobile data plan,
    known as CPID. In some cases it might be possible to use the subscribers
    MSISDNs as their identities. In that case, any additional mechanism to
    establish a separate identity is not necessary. Different applications can
    decide whether to use MSISDN or not, depending on their privacy
    requirements.
1.  Mechanism to query information about activated data plans.
1.  Mechanism to query the user's account details.
1.  Mechanism to query the user's purchased data plans (some may not be
    activated yet).
1.  Mechanism to query the user's eligible upsell offers.
1.  Mechanism to make changes to the user's data plan (e.g., purchase a new
    plan).

The paragraphs that follow elaborate on each of these API components (Sections
[5.1](#5-1-establishing-data-plan-key-string) to [5.6](#5-6-data-purchase) cover
each API part respectively). Unless explicitly noted, all communications MUST
happen over HTTPS (with a valid DPA SSL certificate). Depending on the actual
business product, an operator MAY choose to implement all or a subset of these
API components.

### 5.1 Establishing Data Plan Key String 

GTAF uses a `data_plan_key_string`, which identifies a subscriber to the
operator, when querying the operator's DPA. This `data_plan_key_string` MAY be
the subscriber's MSISDN. However for privacy policy reasons some Google
applications are not able to use MSISDNs and instead need to establish an
alternate identity. We introduce Carrier Plan Identifier (CPID), which
identifies the user's data plan from Google's perspective. In this sense the
CPID is analogous to the MSISDN.

Next, we explain the mechanism that establishes a CPID.

#### 5.1.1 CPID Query 

![CPID](https://github.com/aterzis-google/data-plan-api/blob/master/Data%20Plan%20API%20CPID.png)

**Figure 2. Call flow to establish and maintain a Carrier Plan ID (CPID).**

Figure 2 illustrates the steps involved in establishing a CPID.

1.  A Google client uses a private API to retrieve the CPID endpoint. The API
    uses the client's public IP address and OPTIONALLY the MNC and MCC of the
    network that the client resides to provide one or more operator-provided
    URLs (i.e., CPID endpoints).
1.  Having received the CPID endpoint in step 1, the client issues an HTTP GET
    request in the second step to establish a CPID. The operator MAY support
    sending the request over HTTPS.

    The ISP MAY also use IP addresses instead of domain names in the CPID URL if
    that is preferable. The IP addresses MAY be in private address space but
    they have to be reachable by Google clients inside the operator's network.

The format of the request is:

```
GET CPID_URL?app={carrier_app_id}
```

The `{carrier_app_id}` query parameter contains the application ID corresponding
to the application that issues the request. Each time the client issues a GET
CPID_URL request, it MUST receive a new CPID. The `carrier_app_id` will be
specified by Google during the endpoint registration process.

The CPID endpoint returns an error code if it cannot serve the request.
Currently defined error cases are:

*   CPID endpoint received CPID request for unknown application identifier.
    The CPID endpoint MAY return error code 400.
*   CPID endpoint received CPID request from user who has not opted in the
    program of sharing data plan information with Google. The CPID endpoint MUST
    return error code 403.
*   CPID endpoint received CPID request from user that is roaming. The CPID
    endpoint MUST return error code 403.

The error response MUST include a JSON object with more information about the
error. The error response body has the following structure:

```
{
 "error": string
 "cause": integer
}
```

See Section [5.2.3](#5-2-3-error-cases) for a description of the `cause` field. 
Clients that receive any 4xx response codes MUST restrict any further use of the
Data Plan API until client app restarts or connection type/roaming status
changes.

Otherwise, the CPID endpoint returns a 200 OK response. The response MUST also
include a JSON object. The format of the JSON is:

```
{
 "cpid": CPID_string,    // CPID proposed by the server
 "ttlSeconds": integer   // CPID will expire after this many seconds
}
```

The RECOMMENDED way for the CPID endpoint to create a CPID is:

```
CPID_string = Base64(AES(MSISDN + carrier_app_id + TimeStamp, secret))
```

The CPID endpoint concatenates the MSISDN, the application id and the expiry
timestamp and encrypts it via AES using `secret`key. The encrypted output is
Base64 encoded. Furthermore, when the CPID is used in a URL, it must be
URL-encoded to handle special characters (/+=) used in Base64.

The resulting token MAY be concatenated with the Mobile Country Code and Mobile
Network code. The latter is to ensure that the DPA can route the back-end
requests to the correct operator instance.

```
CPID_string = Base64(AES(MSISDN + carrier_app_id + TimeStamp, secret)) + mcc + mnc
```

The mobile country code consists of 3 decimal digits and the mobile network code
consists of 2 or 3 decimal digits.

#### 5.1.2 Data Plan Key Types

Given that the `data_plan_key_string` MUST be either a CPID or a MSISDN, the
GTAF MUST include the plan key type when querying for data plan information so
that DPA knows how to handle the `data_plan_key_string` properly. We therefore
define the following data plan key types `{CPID, MSISDN}` and a
`key_type` query parameter will be added as part of the URL
string used by GTAF when calling the DPA.

### 5.2 Querying Data Plan Status

#### 5.2.1 GTAF-DPA Interaction

![DPA](https://github.com/aterzis-google/data-plan-api/blob/master/Data%20Plan%20API%20DPA.png)

**Figure 3. Call flow to request and receive user data plan information.**

Figure 3 illustrates the call flow associated with a client querying about the
user's data plan status and other data plan information. This call flow is
shared for the rest of API components
(Sections [5.2](#5-2-querying-data-plan-status) to [5.6](#5-6-data-purchase)).

1.  The client requests **data plan status** and/or **data plan information** by
    calling a private Google API. The client includes the data plan key, be it
    MSISDN or CPID, in the request to the GTAF.
1.  The GTAF uses the data plan key to query the operator's DPA. The GTAF issues
    the following requests
    1.  Information for current data plan, if the client requested the **data
        plan status**.
    1.  Information for current plans and possible upsell plans, if the client
        requested the **data plan information**.
1.  The GTAF returns the requested information to the client.

Steps 1 and 3 in Figure 3 are private Google APIs and are therefore not further
described. Step 2 is a public API described hereafter (Sections
[5.2](#5-2-querying-data-plan-status) to [5.6](#5-6-data-purchase)). The DPA
MUST respect the `Cache-Control: no-cache` HTTP header when serving these API calls from
GTAF.

Before the GTAF can call the DPA it needs to establish a bidirectional security
association with the DPA. As part of operator onboarding process, we will check
the validity of DPA SSL certificate. We currently REQUIRE the use of OAuth for
mutual authentication.

All timestamp fields defined below MUST follow the extended ISO 8601 profile 
defined in RFC 3339. 

#### 5.2.2 Data Plan Status

The GTAF issues the following URL to get the data plan status:

```
GET DPA_URL/{data_plan_key_string}/dataPlanStatus?key_type={CPID,MSISDN}&appid={carrier_app_id}
```

where the `appid=carrier_app_id` query parameter identifies the application
making the request. The query parameter is not included If the application is
unknown. The `carrier_app_id` string is the same one the application
uses to establish the CPID, described in Section [5.1.1](#65-1-1-cpid-query).

The format of the response JSON object is as follows:

```
{
 "dataPlanStatus":
 [
  {
   "planName": string,          // string describing the plan, may
                                // get surfaced to users for some
                                // google apps (opt.)
   "planId": string,            // Plan identifier. Can be used to
                                // refer to the plan during
                                // upsells, etc. (req.)
   "expirationTime": string     // in timestamp format. (req.)
   "planModuleStatus" : [       // (req.)
    {
     "quotaBytes": integer,     // package quota in bytes (opt.)
                                // for unlimited plan, set to 2^63-1.
     "remainingBytes": integer, // byte-based plan balance.
                                // Not used for unlimited plans (opt.)
     "remainingBalanceLevel": string,
                                // coarse account balance level (opt.)
                                // Possible values:
                                // Zero bytes left in the plan module. 
                                // "REMAINING_DATA_ZERO"
                                // Can be used to show an "almost out of                      
                                // data" promotion.
                                // "REMAINING_DATA_LOW",
                                // Data level is high, do nothing.                                                                                             // "REMAINING_DATA_HIGH"
                                // Exactly one of the "remainingBytes",
                                // "remainingTime", and "balanceLevel" 
                                // should be included in the response.
     "pmtcs": [                 // PMTCs that this module applies (req.)
       string                   // e.g., "VIDEO_BROWSING"
     ]
     "priority": integer,       // priority among plan modules across
                                // all dataPlanStatus returned (opt.)
     "quotaMinutes": integer,   // package quota in minutes (opt.)
     "expirationTime": string   // in ISO extended format. (opt.)
     "overusagePolicy": string, // THROTTLED, BLOCKED, PAY_AS_YOU_GO (opt.)
     "currentMaxRate": integer, // in kbits per second (opt.)
     "remainingTime": integer,  // time-based remaining
                                // usage, in minutes (opt.)
     "description": string      // string describing plan module,
                                // maybe surfaced to user and should be
                                // close to market description (opt.)
     "flexTimeWindows": [       // flex time rate window (opt.)
       {"recurrenceType": string// DAILY, DAILY_EXCEPT_WEEKEND,
                                // WEEKEND. (req.)
        "startTime": string,    // timestamp format w/o date. (req.)
        "endTime": string,      // timestamp format w/o date. (req.)
        "accessBlocked": bool   // a boolean indicating whether Internet
                                // access is blocked (opt.)
        "discountPercentage": integer
                                // 0-100 normalized by highest rate (opt.)
        "dataCreditPercentage": integer
                                // Percentage of data used in this time
                                // that will be credited back to users
                                // 0 - 100 (opt.)
       }
     ]
    }
   ]
   …
  },
  …
 ],
 "responseStaleTime": string,   // Indicates when dataPlanStatus
                                // information becomes invalid.
                                // In timestamp format. (opt.)
}
```

Note that one of `remainingBytes`, `remainingBalanceLevel`, or `remainingTime` 
fields MUST be present in `planModuleStatus`.

The request SHALL include an `Accept-Language` header indicating the language
that the human readable strings (e.g., plan descriptions) should be in.

For post-paid plans, `expirationTime` MUST be the plan recurrence date (i.e.,
when data balance gets refreshed/reloaded).

Each plan module may contain multiple Plan Module Traffic Category (`PMTCs)`to
model the case where a plan module is shared among multiple apps (e.g., 500 MB
for game and music). The following PMTCs are pre-defined:

```
GENERIC, VIDEO, VIDEO_BROWSING, MUSIC, GAMING, SOCIAL, MESSAGING and PMTC_UNSPECIFIED.
```

It is expected that mobile operators will contact individual Google teams to
agree on the set of PMTCs and their semantics that are relevant for different
Google applications.

Finally, we introduce `flexTimeWindows` to represent offpeak data plans, or more
general, rate-varying data plan based on time of the day. For example, if a data
plan discounts data usage by 50% everyday from 2 to 6 am, the `flexTimeWindow`
representation would look like: `"flexTimeWindows": [ { "recurrenceType":
"DAILY", "startTime": "02:00:00Z", "endTime": "06:00:00Z", "discountPercentage":
50 } ]`

The absence of `flexTimeWindows`implies that the same charging rate is applied
throughout the lifetime of a data plan.

#### 5.2.3 Error Cases

Currently defined error cases are:

*   User is currently roaming and DPA query is disabled for this user. The DPA
    returns a 403 error.
*   The DPA returns a 404 NOT_FOUND error code indicating to the GTAF that the
    `data_plan_key_string` is invalid (i.e., non-existing CPID/MSISDN).
*   The DPA returns a 410 GONE error code indicating to the GTAF that the client
    should get a new CPID if *key_type = CPID* and the CPID has expired.
*   The DPA returns a 501 NOT_IMPLEMENTED error code indicating that it does 
    not support this call.
*   Service temporarily unavailable. The DPA returns a 503 SERVICE UNAVAILABLE 
    with the Retry-After header indicating when a new request can be attempted.
*   The DPA returns a 500 INTERNAL SERVER ERROR error code for all other
    unspecified errors.

In all error cases, the DPA MUST include a JSON object with more information
about the error. The error response body MUST use the following structure:

```
{
  "error": string,
  "cause": integer,
}

```

The following `cause` values are currently defined:

1.  INVALID_NUMBER = 1. The phone number provided in the request is
    invalid. This error cause should be used for all requests that failed due to
    a bad phone number (e.g., the 404 NOT FOUND error mentioned above, as well
    as the 404 error in Section[5.6.2](#5-6-2-purchase-response)).
1.  INCOMPATIBLE_PLAN = 2. The plan included in the request (e.g., data
    purchase, see Section [5.6](#5-6-data-purchase)) is incompatible with the
    user.
1.  DUPLICATE_TRANSACTION = 3. The requested transaction is a duplicate of a
    previous transaction (see Section [5.6](#5-6-data-purchase)).
1.  BAD_REQUEST = 4. Original request was malformed.
1.  BAD_CPID = 5. The CPID provided in the request has expired or is otherwise
    unrecognized.
1.  BACKEND_FAILURE = 6. While the DPA itself is operational, the backend
    necessary to complete the request is not.
1.  REQUEST_QUEUED = 7. The DPA has queued a request for processing but the 
    outcome of the request is still pending. The client may retry the request
    to retrieve the result of the request. The DPA will return REQUEST_QUEUED
    to requests that arrive before processing of the original request has completed.
1.  UNKNOWN_APP = 8. The CPID endpoint received a request from an unknown application. 
1.  USER_ROAMING = 9. The CPID endpoint or the DPA received a request for a user
    that is currently roaming. 
1.  USER_OPT_OUT = 10. User has not opted in the program.
1.  SIM_RELOAD_REQUIRED = 11. The phone number provided in the request is in a
    "grace period" where a subscriber can receive incoming calls but cannot use
    mobile data. User has to top up credits or buy data packs to get out of this state.

Otherwise, the DPA returns a 200 OK. We note that these `cause` values are used
for all responses.

### 5.3 Querying User Account

The GTAF issues the following URL to get user's account information:

```
GET DPA_URL/{data_plan_key_string}/account?key_type={MSISDN,CPID}&appid={carrier_app_id}
```

The format of the response JSON object is as follows:

```
{
 "account":
  {
   "remainingWalletBalance": string,   // e.g., "100.1" (opt.)
   "costCurrency": string,              // e.g., baht (opt.)
   "accountType": string,               // PREPAID, POSTPAID (opt.)
   "responseStaleTime": string, // Indicates when user account
                                // information becomes invalid.
                                // In timestamp format. (opt.)
  }
}
```

The error cases and causes for the account API call are the same as the ones
defined in Section [5.2.3](#5-2-3-error-cases).

### 5.4 Querying Purchased Plans

The GTAF issues the following URL to get user's purchased plans:

```
GET DPA_URL/{data_plan_key_string}/purchasedPlans?key_type={MSISDN,CPID}&appid={carrier_app_id}
```

The format of the response JSON object is as follows:

```
{
 "purchasedPlans": [
   {
    "planName": string,          // string describing the plan (opt.)
    "planId": string,            // Plan identifier. Can be used to refer
                                 // to the plan during upsells, etc. (req.)
    "planDescription": string,   // string describing the plan (opt.)
    "promoMessage": string,      // string describing the plan (opt.)
    "planLanguage": string,      // two letter ISO language-code (opt.)
    "cost": string,              // (opt.)
    "costCurrency": string,      // (opt.)
    "connectionType": string,    // CONNECTION_2_G, CONNECTION_3_G,
                                 // CONNECTION_4_G, CONNECTION_ALL
                                 // (all connection types). (opt.)
    "expirationTime": string,    // in timestamp format. (req.)
    "planModuleStatus" : [       // (req.)
    {
     "remainingBytes": integer,  // (opt.)
     "pmtcs": [                  // pmtcs that this module applies (req.)
       string,                   // e.g., "MUSIC", "GAME"
     ]
     "expirationTime": string    // in timestamp format. (opt.)
     "overusagePolicy": string,  // THROTTLED, BLOCKED,
                                 // PAY_AS_YOU_GO (opt.)
     "currentmaxRate": integer,  // in kbits per second (opt.)
     "remainingTime": integer,   // time-based remaining
                                 // usage, in minutes (opt.)
     "remainingBalanceLevel": string,
                                // coarse account balance level (opt.)
                                // Possible values:
                                // Zero bytes left in the plan module. 
                                // "REMAINING_DATA_ZERO"
                                // Can be used to show an "almost out of                      
                                // data" promotion.
                                // "REMAINING_DATA_LOW",
                                // Data level is high, do nothing.                                                                                             // "REMAINING_DATA_HIGH"
                                // Exactly one of the "remainingBytes",
                                // "remainingTime", and "balanceLevel" 
                                // should be included in the response.                                 
     "quotaBytes": integer,      // package quota in bytes (opt.)
                                 // for unlimited plan, set to 2^63-1.
     "quotaMinutes": integer,    // package quota in minutes (opt.)
     "description": string,      // string describing plan module,
                                 // maybe surfaced to user and should be
                                 // close to market description (opt.)
     "flexTimeWindows": [        // flex time rate rate window (opt.)
     {
      "recurrenceType": string,  // DAILY, DAILY_EXCEPT_WEEKEND,
                                 // WEEKEND (req.)
      "startTime": string,       // timestamp format w/o date. (req.)
      "endTime": string,         // timestamp format w/o date. (req.)
      "accessBlocked": bool      // is access blocked (opt.)
      "discountPercentage": integer
                                 // 0-100 normalized by highest rate (opt.)
      "dataCreditPercentage": integer
                                 // Percentage of data used in this time
                                 // that will be credited back to users
                                 // 0 - 100 (opt.)
       }
     ]
    }
   ]
  …
   },
   …
  ],
 "responseStaleTime": string,   // Indicates when purchased plan
                                // information becomes invalid.
                                // In timestamp format. (opt.)
}
```

Note that one of `remainingBytes`, `remainingBalanceLevel`, or `remainingTime` 
field MUST be present in planModule.

The request SHALL include an `Accept-Language` header indicating the language
that the human readable strings (e.g., plan descriptions) should be in. The
error cases and causes for the purchased plans API call are the same as the ones
defined in [Section 5.2.3](#5-2-3-error-cases).

### 5.5 Querying Upsell Offer

The GTAF issues the following URL to get the upsell offer information:

```
GET DPA_URL/{data_plan_key_string}/upsellOffer?key_type={MSISDN,CPID}&appid={carrier_app_id}&context={purchase_context}
```

The optional context parameter provides the application context the request is
made in.

The format of the response JSON object is as follows:

```
{
 "upsellOffer": {
  "upsellInfo": {
   "upsellPromoMessage": string,        // describing why the subscriber
                                        // should buy (opt.)
   "moreInformation_message" : string,  // Allow the user to read more
                                        // about the upsell offers. (opt.)
   "moreInformationUrl" : string,       // URL to page with  opt-out from
                                        // offers inside the
                                        // application. (opt.)
   "carrierBrandName": string,          // mobile carrier brand name, may
                                        // differ from prepaid to
                                        // postpaid (req.)
   "carrierLogoImageUrl": string,       // URL of the logo image of mobile
                                        // carrier (req.)
  },
  "upsellPlans": [
   {
    "planName": string,          // string describing the plan (req.)
    "planId": string,            // Plan identifier. Can be used to refer
                                 // to the plan during upsells, etc. (req.)
    "planDescription": string,   // string describing the plan (req.)
    "promoMessage": string,      // string describing the plan (opt.)
    "promoImage": string,        // URL to promo logo (opt.)
    "planLanguage": string,      // two letter ISO language-code (opt.)
    "overusagePolicy": string,   // THROTTLED, BLOCKED,
                                 // PAY_AS_YOU_GO (opt.)
    "pmtcs": [ 
      string,                    // pmtcs that this module applies, e.g.,
    ]                            // VDP, GENERIC, COMPOSITE, etc. (req.)
    "maxRate": integer           // current rate (opt.)
    "cost": string               // (req.)
    "costCurrency": string       // (req.)
    "connectionType": string,    // CONNECTION_2_G, CONNECTION_3_G,
                                 // CONNECTION_4_G, CONNECTION_ALL
                                 // (all connection types). (req.)
    "duration": integer,         // plan validity period in seconds (req.)
    "quotaBytes": integer,       // package quota in bytes (req.)
                                 // for unlimited plan, set to 2^63-1.
    "quotaMinutes": integer,     // package quota in minutes (opt.)
    "upsellOfferContext": string,// required, used to identify offer.
    …
   },…
  ],
  "responseStaleTime": string,   // Indicates when upsell offer
                                 // information becomes invalid.
                                 // In timestamp format. (opt.)
 }
}
```

The order of the upsell plan(s) in the `upsellPlans` array SHALL determine the
order in which the upsell plan(s) are presented to users. Further, if the
application can present only *x* plans due to UI or other limitations and the
response contains *y > x* upsell plans only the first *x* upsell plans SHALL be
presented. If an upsell plan costs more than user's remaining wallet balance,
such upsell plan will not be presented to the user.

The strings in `upsellInfo` are intended to allow the user to read more about
the upsell offers, and also include a way to opt-out from receiving more offers
from inside applications. The reason for having these fields is that some
operators do not need end-user consent to allow in-app upsells but rather
require a mechanism for users to opt-out. Note that if DPA can not provide
account information, any upsell plan returned MUST not cost more than user's
wallet balance.

The request SHALL include an `Accept-Language` header indicating the language
that the human readable strings (e.g., plan descriptions) should be in. The
error cases and causes for the upsell offer API call are the same as the ones
in [Section 5.2.3](#5-2-3-error-cases).

### 5.6 Data Purchase 

The purchase plan API defines how to purchase data plans by GTAF. The GTAF
initiates the transaction to purchase one data plan to the DPA. The request
SHALL include a unique transaction identifiers (transactionId) to trace requests
and avoid duplicate transaction execution. The DPA MUST respond with a
success/failure response.

#### 5.6.1 Purchase Request

Once it receives a request from a client, the GTAF issues a POST request to the
DPA. The URL of the request is:

```
POST DPA_URL/data_plan_key_string/purchasePlan?key_type={CPID,MSISDN}&appid={carrier_app_id}
```

where `data_plan_key_string` is either `cpid_string` or `MSISDN` (if per50mitted).
The body of the request includes the following fields:

```
{
 "purchaseRequest":{
   "planId": string,        // Id of plan to be purchased. Copied from
                            // upsellPlans.planId field returned from a
                            // Upsell Offer request,
                            // if available. (req.).
   "transactionId": string, // Unique request identifier (req.)
   "offerContext": string   // Copied from from the
                            // upsellPlans.upsellOfferContext, if
                            // available. (opt.)
  }
}
```

#### 5.6.2 Purchase Response 

The DPA SHALL return the error codes and causes defined in 
Section [5.2.3](#5-2-3-error-cases). Additionally, the following error 
codes are defined to represent failed transaction outcomes:

*   The DPA returns a 400 BAD REQUEST error code indicating to the GTAF that the
    purchased plan ID is invalid.
*   The DPA returns a 402 PAYMENT REQUIRED error code indicating to the GTAF
    that user does not have sufficient balance to complete the purchase.
*   The DPA returns a 403 FORBIDDEN error code indicating to the GTAF that the
    current transaction is a duplicate of a previously succeed transaction.
*   The DPA returns a 409 CONFLICT error code indicating to the GTAF that the
    plan to be purchased is incompatible with user's current product mix. For
    example, if the carrier data plan policy disallows mixing postpaid and
    prepaid plans, attempting to purchase a prepaid plan for a postpaid user
    will therefore lead to a 409 CONFLICT error.

The DPA SHALL only generate a 200-OK response for a successfully executed
transaction. The body of the response includes the following transaction
details:

```
{
  "purchaseResponse": {
    "planId": string,               // copied from request. (req.)
    "transactionId": string,        // copied from request. (req.)
    "transactionMessage": string,   // status message. (opt.)
    "confirmationCode": string,     // DPA-generated confirmation code
                                    // for successful transaction. (opt.)

    "planActivationTime" : string,  // Time when plan will be activated,
                                    // in timestamp format. (opt.)
  },
 // walletInfo is optionally populated if DPA can get the updated balance
 // after the transaction is completed. Otherwise, client will call
 // the account endpoint to get the updated balance.
 "walletInfo": {
  "remainingWalletBalance": string, // e.g., "100.33" (opt.)
  "costCurrency": string,           // ISO4217 format (opt.)
 }
}
```

If the `planActivationTime` is missing, GTAF SHALL assume that the plan has been
activated.

### 5.7 Eligibility

The GTAF may issue the following eligibility request to check whether a user,
identified by CPID or MSISDN, is eligible to purchase a plan.

```
GET DPA/{data_plan_key_string}/Eligibility/{plan_id}?key_type={MSISDN,CPID}
```

Note that `plan_id` is the unique identifier for the plan that can be used to
purchase the plan on behalf of the user (See
Section [5.6](#5-6-data-purchase)). If `plan_id` is not specified the DPA MAY
choose to return all plans purchasable by that user.

The DPA SHALL return the error codes and causes defined in 
Section [5.2.3](#5-2-3-error-cases). Additionally, the DPA SHALL return 
an error in the following error cases:

*   The DPA returns a 400 BAD REQUEST error code indicating to the GTAF that
    `plan_id` is invalid.
*   The DPA returns a 409 CONFLICT error code indicating that `plan_id` is
    incompatible with the user's data plan.

Otherwise, the DPA SHALL return a 200-OK response. The format of a 
successful response is:

```
{
  "eligiblePlans":
  [
   {
    "planId": string,   // Plan identifier. Can be used to
                        // refer to the plan during
                        // upsells, etc. (req.)
   }
  ]
}
```

When the request includes a `plan_id` the response includes only that
plan. Otherwise, the list includes all the plans the user is eligible to
purchase. In the case where `plan_id` is empty and the DPA does not support
returning the list of eligible plans it MUST return a 400 BAD REQUEST error.

<!-- Footnotes themselves at the bottom. -->

## Notes

[^1]: The VIDEO_OFFLINE PMTC means this plan is good for offline only (e.g.,
    really bad streaming QoE). It is independent of FlexTime window.
