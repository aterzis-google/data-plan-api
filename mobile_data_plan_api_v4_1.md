## Public Data Plan API v4.1

### gtaf-eng@google.com

### August 9th, 2016

Table of Contents

[TOC]

## 1. Motivation {#1-motivation}

The goal of this document is to describe the public interfaces that Google will
use to identify the users' mobile data plans, collect information about these
plans, and purchase data plans.

## 2. Changes {#2-changes}

The v4.1 API is entirely **backwards compatible** with v4 and previous version
of this API. The v4.1 API introduces the following changes:

*   In Section [5.1.1](#5-1-1-cpid-query), we introduced a new error code (400),
    generated when the CPID endpoint receives a request with an unknown
    application identifier.
*   In Section [5.2.2](#5-2-2-data-plan-status), we clarified that the `roaming`
    field in the `dataPlanStatus` response is a boolean.
*   In Section [5.2.2](#5-2-2-data-plan-status), we deprecated the
    `responseStaleTime` field in `dataPlanStatus`. Instead we use the
    `responseStaleTime`field at the end of the response. New implementations
    MUST not include this field in the responses they generate. Existing
    implementations can continue generating this field. In a future version of
    the specification this fields will be removed completely and WILL NOT be
    accepted.
*   In Section [5.2.2](#bookmark=id.6sy3tpn3vrov), `planModuleStatus` is now a
    required field.
*   In Section [5.2.2](#bookmark=id.6sy3tpn3vrov), we fixed a typo (`planModule`
    should have been `planModuleStatus).`
*   In Sections [5.2.2](#bookmark=id.6sy3tpn3vrov),
    [5.4](#5-4-querying-purchased-plans), and [5.5.1](#5-5-1-data-plan-info) we
    deprecated the following fields in the `dataPlanStatus` and `purchasedPlans`
    objects: `planType, currentMaxRate/maxRate, overusagePolicy, quotaBytes,
    quotaMinutes, remainingBytes, remainingTime.` Instead, we use the
    corresponding fields inside `planModuleStatus` for the same purpose. New
    implementations MUST not include these fields in the responses they
    generate. Existing implementations can continue generating these fields. In
    a future version of the specification these fields will be removed
    completely and WILL NOT be accepted.
*   In Section [5.4](#5-4-querying-purchased-plans) and
    [5.5.1](#5-5-1-data-plan-info), the `planModules` field has been renamed to
    `planModuleStatus` and is now a required field. In the same sections, the
    field `maxRate` in `planModuleStatuss` has been renamed to `currentMaxRate`
    for consistency.
*   In Sections [5.4](#5-4-querying-purchased-plans) and
    [5.5.1](#5-5-1-data-plan-info), we clarified that `discountPercentage` is an
    integer.
*   In Section [5.5](#bookmark=id.vg8ljjbmqnks) and
    [5.5.1](#5-5-1-data-plan-info) we clarified that the following fields in the
    `upsellInfo`and`upsellPlans`objects` `are required: `carrierBrandName,
    carrierLogoImageUrl, planName, planDescription, planType, cost,
    costCurrency, connectionType, quotaBytes, upselOfferContext.`
*   In Sections [5.5](#5-5-querying-upsell-offer) and
    [5.5.1](#5-5-1-data-plan-info) we clarified that the `moreInformationUrl`
    field is optional.
*   In Section [5.5.1](#5-5-1-data-plan-info), we added the `accessblocked`
    field to match Sections [5.2.2](#5-2-2-data-plan-status) and
    [5.4](#5-4-querying-purchased-plans).

## 3. Terminology {#3-terminology}

This document uses the following terms:

*   **CPID.** Carrier Plan Identifier. An opaque string that has a many-to-one
    correspondence with a user's MSISDN.
*   **DPA.** Data Plan Agent. An operator function that returns information
    about users' data plans and account details. The DPA can optionally generate
    CPIDs and make changes to users' data plans (e.g., purchase new plans).
*   **GTAF.** Google Traffic Application Function. A Google function that
    interacts with DPA on behalf of all Google apps.

### 3.1 Requirements Language {#3-1-requirements-language}

The keywords "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).

## 4. Data Plan Terminology {#4-data-plan-terminology}

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
        following PMTCs : `GENERIC, VIDEO, VIDEO_BROWSING, VIDEO_OFFLINE`[^2]`,
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

![Data
Plan](https://docs.google.com/drawings/d/1GFO0mpcQC7NFjWwzSSY_t3Jwnwx-JA2nrXUIO3hDITU/export/png)

[view/edit](https://docs.google.com/drawings/d/1GFO0mpcQC7NFjWwzSSY_t3Jwnwx-JA2nrXUIO3hDITU/edit)

**Figure 1. Sample data plans.**

## 5. API Description {#5-api-description}

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

### 5.1 Establishing Data Plan Key String {#5-1-establishing-data-plan-key-string}

GTAF uses a `data_plan_key_string`, which identifies a subscriber to the
operator, when querying the operator's DPA. This `data_plan_key_string` MAY be
the subscriber's MSISDN. However for privacy policy reasons some Google
applications are not able to use MSISDNs and instead need to establish an
alternate identity. We introduce Carrier Plan Identifier (CPID), which
identifies the user's data plan from Google's perspective. In this sense the
CPID is analogous to the MSISDN.

Next, we explain the mechanism that establishes a CPID.

#### 5.1.1 CPID Query {#5-1-1-cpid-query}

![CPID](https://docs.google.com/drawings/d/1vGKmdwyVFf0mUfGRh7ithNQvxr2oQ6gmzuWEbMeZ710/export/png)

[view/edit](https://docs.google.com/drawings/d/1vGKmdwyVFf0mUfGRh7ithNQvxr2oQ6gmzuWEbMeZ710/edit)

**Figure 2. Call flow to establish and maintain a Carrier Plan ID (CPID).**

Figure 2 illustrates the steps involved in establishing a CPID ([Section
6.1](#6-1-endpoint-registration-process) discusses the mechanism that operators
use to provide GTAF with CPID endpoint details).

1.  A Google client uses a private API to retrieve the information stored during
    the registration process described in Section
    [6.1](#6-1-endpoint-registration-process). The API uses the client's public
    IP address and OPTIONALLY the MNC and MCC of the network that the client
    resides to provide one or more operator-provided URLs (i.e., CPID
    endpoints).
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

The {carrier_`app_id}` query parameter contains the application ID corresponding
to the application that issues the request. Each time the client issues a GET
CPID_URL request, it MUST receive a new CPID. The `carrier_app_id` will be
specified by mobile operator during the endpoint registration process (see
[Section 6.1](#6-1-endpoint-registration-process)).

The CPID endpoint returns an error code if it cannot serve the request.
Currently defined error cases are:

*   CPID endpoint received CPID request from user who has not opted in the
    program of sharing data plan information with Google. The CPID endpoint MUST
    return error code 403.
*   CPID endpoint received CPID request from user that is roaming. The CPID
    endpoint MUST return error code 403.
*   **CPID endpoint received CPID request for unknown application identifier.
    The CPID endpoint MUST return error code 400.**

The error response MUST include a JSON object with more information about the
error. The error response body has the following structure:

```
{
 "error": string
}
```

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

#### 5.1.2 Data Plan Key Types {#5-1-2-data-plan-key-types}

Given that the `data_plan_key_string` MUST be either a CPID or a MSISDN, the
GTAF MUST include the plan key type when querying for data plan information so
that DPA knows how to handle the `data_plan_key_string` properly. We therefore
define the following data plan key types **<code>{CPID, MSISDN}</code>** and a
**<code>key_type</code>** query parameter will be added as part of the URL
string used by GTAF when calling the DPA.

### 5.2 Querying Data Plan Status {#5-2-querying-data-plan-status}

#### 5.2.1 GTAF-DPA Interaction {#5-2-1-gtaf-dpa-interaction}

![DPA](https://docs.google.com/drawings/d/11fJuPf4uW3_Se8gJoRz7dY1b8RdcG-SnBYrXykZbEGg/export/png)

[view/edit](https://docs.google.com/drawings/d/11fJuPf4uW3_Se8gJoRz7dY1b8RdcG-SnBYrXykZbEGg/edit)

**Figure 3. Call flow to request and receive user data plan information.**

Figure 3 illustrates the call flow associated with a client querying about the
user's data plan status and other information ([Section
6.1](#6-1-endpoint-registration-process) discusses the mechanism that operators
use to provide GTAF with DPA endpoint details). This call flow is shared for the
rest of API components (Sections [5.2](#5-2-querying-data-plan-status) to
[5.6](#5-6-data-purchase)).

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
MUST respect the "**<code>cache-control"</code>** HTTP header when serving these
API calls from GTAF.

Before the GTAF can call the DPA it needs to establish a bidirectional security
association with the DPA. As part of operator onboarding process, we will check
the validity of DPA SSL certificate. We currently REQUIRE the use of OAuth for
mutual authentication.

#### 5.2.2 Data Plan Status {#5-2-2-data-plan-status}

The GTAF issues the following URL to get the data plan status:

`GET DPA_URL/{data_plan_key_string}/dataPlanStatus?key_type={CPID,MSISDN}`

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
   "planType": string,          // (DEPRECATED)
   "planCategory": string,      // PREPAID, POSTPAID (opt.)
   "quotaBytes": integer,       // (DEPRECATED)
   "quotaMinutes": integer,     // (DEPRECATED)
   "expirationTime": string     // in ISO8601 extended format. (req.)
   "overusagePolicy": string,   // DEPRECATED
   "currentMaxRate": integer,   // (DEPRECATED)
   "remainingBytes": integer,   // (DEPRECATED)
   "remainingTime": integer,    // (DEPRECATED)
                                // usage, in minutes (opt.)
   "roaming": boolean,          // boolean indicating roaming state (opt.)
   "responseStaleTime": string, // (DEPRECATED)
   "planModuleStatus" : [       // (req.)
    {
     "quotaBytes": integer,     // package quota in bytes (opt.)
                                // for unlimited plan, set to 2^63-1.
     "remainingBytes": integer, // byte-based plan balance.
                                // Not used for unlimited plans (opt.)
     "pmtcs": [                 // PMTCs that this module applies (req.)
       string                   // e.g., "VIDEO_BROWSING"
     ]
     "priority": integer,       // priority among plan modules across
                                // all dataPlanStatus returned (opt.)
     "quotaMinutes": integer,   // package quota in minutes (opt.)
     "expirationTime": string   // in ISO8601 extended format. (opt.)
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
        "startTime": string,    // ISO 8601 extended w/o date. (req.)
        "endTime": string,      // ISO 8601 extended w/o date. (req.)
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
                                // In ISO8601 extended format. (opt.)
}
```

Note that at least one of `remainingBytes` or `remainingTime` fields MUST be
present in **<code>planModuleStatus</code>**.

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
Google applications. Alternatively, we introduce a data plan metadata
registration mechanism that helps to ease this process ([Section
6.2](#6-2-data-plan-metadata-registration-process)).

Finally, we introduce `flexTimeWindows` to represent offpeak data plans, or more
general, rate-varying data plan based on time of the day. For example, if a data
plan discounts data usage by 50% everyday from 2 to 6 am, the `flexTimeWindow`
representation would look like: `"flexTimeWindows": [ { "recurrenceType":
"DAILY", "startTime": "02:00:00Z", "endTime": "06:00:00Z", "discountPercentage":
50 } ]`

The absence of `flexTimeWindows`implies that the same charging rate is applied
throughout the lifetime of a data plan.

#### 5.2.3 Error Cases {#5-2-3-error-cases}

Currently defined error cases are:

*   The DPA returns a 404 NOT_FOUND error code indicating to the GTAF that the
    `data_plan_key_string` is invalid (i.e., non-existing CPID/MSISDN).
*   The DPA returns a 410 GONE error code indicating to the GTAF that the client
    should get a new CPID if *key_type = CPID* and the CPID has expired.
*   User is currently roaming and DPA query is disabled for this user. The DPA
    returns a 403 error.

In all error cases, the DPA MUST include a JSON object with more information
about the error. The error response body MUST use the following structure: `{
"error": string }`

Otherwise, the DPA returns a 200 OK.

### 5.3 Querying User Account {#5-3-querying-user-account}

The GTAF issues the following URL to get user's account information:

```
GET DPA_URL/{data_plan_key_string}/account?key_type={MSISDN,CPID}
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
                                // In ISO8601 extended format. (opt.)
  }
}
```

The error cases for account API call are the same as the ones defined in Section
[5.2.3](#5-2-3-error-cases).

### 5.4 Querying Purchased Plans {#5-4-querying-purchased-plans}

The GTAF issues the following URL to get user's purchased plans:

```
GET DPA_URL/{data_plan_key_string}/purchasedPlans?key_type={MSISDN,CPID}
```

The format of the response JSON object is as follows[^3]:

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
    "overusagePolicy": string,   // DEPRECATED
    "planType": string,          // DEPRECATED
    "quotaBytes": integer,       // DEPRECATED
    "quotaMinutes": integer,     // DEPRECATED
    "maxRate": integer,          // DEPRECATED
    "cost": string,              // (opt.)
    "costCurrency": string,      // (opt.)
    "connectionType": string,    // CONNECTION_2_G, CONNECTION_3_G,
                                 // CONNECTION_4_G, CONNECTION_ALL
                                 // (all connection types). (opt.)
    "expirationTime": string,    // in ISO8601 extended format. (req.)
    "remainingBytes": integer    // DEPRECATED
    "remainingMinutes": integer  // DEPRECATED
    "planModuleStatus" : [       // (req.)
    {
     "remainingBytes": integer,  // (opt.)
     "pmtcs": [                  // pmtcs that this module applies (req.)
       string,                   // e.g., "MUSIC", "GAME"
     ]
     "expirationTime": string    // in ISO8601 extended format. (opt.)
     "overusagePolicy": string,  // THROTTLED, BLOCKED,
                                 // PAY_AS_YOU_GO (opt.)
     "currentmaxRate": integer,  // in kbits per second (opt.)
     "remainingTime": integer,   // time-based remaining
                                 // usage, in minutes (opt.)
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
      "startTime": string,       // ISO 8601 extended w/o date. (req.)
      "endTime": string,         // ISO 8601 extended w/o date. (req.)
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
                                // In ISO8601 extended format. (opt.)
}
```

Note that at least one of `remainingBytes` or `remainingTime` field MUST be
present in planModule.

The request SHALL include an `Accept-Language` header indicating the language
that the human readable strings (e.g., plan descriptions) should be in. The
error cases for purchased plans API call are the same as the ones defined in
[Section 5.2.3](#5-2-3-error-cases).

### 5.5 Querying Upsell Offer {#5-5-querying-upsell-offer}

The GTAF issues the following URL to get the upsell offer information:

```
GET DPA_URL/{data_plan_key_string}/upsellOffer?key_type={MSISDN,CPID}
```

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
    "planType": string,          // VDP, GENERIC, COMPOSITE, etc. (req.)
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
                                 // In ISO8601 extended format. (opt.)
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
error cases for the upsell offer API call are the same as the ones in [Section
5.2.3](#5-2-3-error-cases).

#### 5.5.1 Data Plan Info {#5-5-1-data-plan-info}

Given that account, purchased plans and upsell offer, are often requested
together, GTAF MAY issue the following URL to get all data plan information at
once:

```
GET DPA_URL/{data_plan_key_string}?key_type={CPID,MSISDN} → all information at once
```

The following JSON response is returned when the request is for "`all`",
includes all three `accountStatus, purchasedPlans,`and`upsellOffer`objects:

```
{
 "account":
  {
   "remainingWalletBalance": string,      // e.g., "100.1" (opt.)
   "costCurrency": string,                // ISO4217 format (opt.)
   "accountType": string                  // PREPAID, POSTPAID (opt.)
  },
 "purchasedPlans": [
   {
    "planName": string,          // string describing the plan (opt.)
    "planId": string,            // Plan identifier. Can be used to refer
                                 // to the plan during upsells, etc. (req.)
    "planDescription": string,   // string describing the plan (req.)
    "promoMessage": string,      // string describing the plan (req.)
    "planLanguage": string,      // two letter ISO language-code (req.)
    "overusagePolicy": string,   // DEPRECATED
    "planType": string,          // DEPRECATED
    "quotaBytes": integer,       // DEPRECATED
    "quotaMinutes": integer,     // DEPRECATED
    "maxRate": integer,          // DEPRECATED
    "cost": integer,             // (req.)
    "costCurrency": string,      // ISO4217 format (req.)
    "connectionType": string,    // CONNECTION_2_G, CONNECTION_3_G,
                                 // CONNECTION_4_G, CONNECTION_ALL
                                 // (all connection types). (req.)
    "expirationTime": string,    // in ISO8601 extended format. (req.)
    "remainingBytes": integer    // DEPRECATED
    "remainingMinutes": integer  // DEPRECATED
    "planModuleStatus" : [       // (req.)
    {
     "remainingBytes": integer,  // (req.)
     "pmtcs": [                  // pmtcs that this module applies (req.)
       "MUSIC",
       "GAME"
     ]
     "expirationTime": string    // in ISO8601 extended format. (opt.)
     "overusagePolicy": string,  // THROTTLED, BLOCKED,
                                 // PAY_AS_YOU_GO (opt.)
     "currentmaxRate": integer,  // in kbits per second (opt.)
     "remainingTime": integer,   // time-based remaining
                                 // usage, in minutes (opt.)
     "quotaBytes": integer,      // package quota in bytes (opt.)
                                 // for unlimited plan, set to 2^63-1.
     "quotaMinutes": integer,    // package quota in minutes (opt.)
     "description": string       // string describing plan module,
                                 // maybe surfaced to user and should be
                                 // close to market description (opt.)
     "flexTimeWindows": [        // flex time rate rate window (opt.)
     {
      "recurrenceType": string,  // DAILY, DAILY_EXCEPT_WEEKEND,
                                 // WEEKEND (req.)
      "startTime": string,       // ISO 8601 extended w/o date. (req.)
      "endTime": string,         // ISO 8601 extended w/o date. (req.)
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
 "upsellOffer": {
  "upsellInfo": {
   "upsellPromoMessage": string,        // describing why the subscriber
                                        // should buy (req.)
   "moreInformation_message" : string,  // Allow the user to read more
                                        // about the upsell offers. (req.)
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
    "planLanguage": string,      // two letter ISO language-code (req.)
    "overusagePolicy": string,   // THROTTLED, BLOCKED,
                                 // PAY_AS_YOU_GO (opt.)
    "planType": string,          // VDP, GENERIC, COMPOSITE, etc. (req.)
    "maxRate": integer           // current rate (req.)
    "cost": integer              // (req.)
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
  ]
 },
 "responseStaleTime": string,   // Indicates when user related
                                // information becomes invalid.
                                // In ISO8601 extended format. (opt.)
}
```

The request SHALL include an `Accept-Language` header indicating the language
that the human readable strings (e.g., plan descriptions) should be in. The
error cases for get all information API call are the same as the ones in
[Section 5.2.3](#5-2-3-error-cases).

### 5.6 Data Purchase {#5-6-data-purchase}

The purchase plan API defines how to purchase data plans by GTAF. The GTAF
initiates the transaction to purchase one data plan to the DPA. The request
SHALL include a unique transaction identifiers (transactionId) to trace requests
and avoid duplicate transaction execution. The DPA MUST respond with a
success/failure response.

#### 5.6.1 Purchase Request {#5-6-1-purchase-request}

Once it receives a request from a client, the GTAF issues a POST request to the
DPA. The URL of the request is:

```
POST DPA_URL/data_plan_key_string/purchasePlan?key_type={CPID,MSISDN}
```

where `data_plan_key_string` is either `cpid_string` or `MSISDN` (if permitted).
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

#### 5.6.2 Purchase Response {#5-6-2-purchase-response}

The DPA SHALL return an error in the following error cases that are unrelated to
the transaction itself:

*   The DPA returns a 404 NOT FOUND error code indicating to the GTAF that the
    `data_plan_key_string` is invalid (i.e., non-existing CPID/MSISDN).
*   The DPA returns a 410 GONE error code indicating to the GTAF that the client
    should get a new CPID if *key_type = CPID* and the CPID has expired.
*   Data plan purchase is temporarily unavailable. The DPA returns a 503 SERVICE
    UNAVAILABLE with the Retry-After header indicating when a new request can be
    attempted.

Additionally, the following error codes are defined to represent failed
transaction outcome:

*   The DPA returns a 409 CONFLICT error code indicating to the GTAF that the
    plan to be purchased is incompatible with user's current product mix. For
    example, if the carrier data plan policy disallows mixing postpaid and
    prepaid plans, attempting to purchase a prepaid plan for a postpaid user
    will therefore lead to a 409 CONFLICT error.
*   The DPA returns a 402 PAYMENT REQUIRED error code indicating to the GTAF
    that user does not have sufficient balance to complete the purchase.
*   The DPA returns a 400 BAD REQUEST error code indicating to the GTAF that the
    purchased plan ID is invalid.
*   The DPA returns a 403 FORBIDDEN error code indicating to the GTAF that the
    current transaction is a duplicate of a previously succeed transaction.
*   The DPA returns a 500 INTERNAL SERVER ERROR error code for all other
    unspecified errors.

The error response MUST include a JSON object with more information about the
error. The error response body has the following structure:

```
{
 "error": string
}
```

The DPA SHALL only generate a 200-OK response for a successfully executed
transaction. The body of the response includes the following transaction
details:

```json
{
  "purchaseResponse": {
    "planId": string,               // copied from request. (req.)
    "transactionId": string,        // copied from request. (req.)
    "transactionMessage": string,   // status message. (opt.)
    "confirmationCode": string,     // DPA-generated confirmation code
                                    // for successful transaction. (opt.)

    "planActivationTime" : string,  // Time when plan will be activated,
                                    // in ISO8601 extended format. (opt.)
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

## 6. Operator-GTAF Registration {#6-operator-gtaf-registration}

Google clients connected through an operator's network need to reach the CPID
endpoint that the operator configures. Likewise, the GTAF needs to know the DPA
endpoint(s) associated with each operator. Operators MAY configure this
information along with other information related to data plans through the
registration process described below.

### 6.1 Endpoint Registration Process {#6-1-endpoint-registration-process}

As a first step, operators can register their endpoints using out of band
mechanism such as email. Google will also provide a web form in the ISP peering
portal (`isp.google.com`) for operators to configure the information.

In the long term, Google will provide an API that operators can use to update
their information programmatically. To use the API, operators MUST register with
Google's ISP portal. This allows Google to verify that the operator's client is
allowed to send and retrieve information on behalf of the mobile network it
represents. Once registered, the operator creates a service account and
registers it with the Google ISP Portal.

The operator uses this account to update the list of endpoints by using the
following RESTful request:

```
PUT https://www.googleapis.com/ispportal/v0/vdp/endpoints
```

In the request body, supply data with the following structure:

```
{
 "mobileCarrierInfo": {
  "networkName": "Friendly Operator", // string with operator name (req.)
  "networkInfo": {
   "asn": integer,                    // operator's autonomous system number
   "countryCode": integer,            // operator's mobile country code (MCC)
   "networkCode": integer             // operator's mobile network code (MNC)
   "planStatusQpsHint": integer       // carrier specified qps limit when
                                      // calling get data plan info API.
   "planInfoQpsHint": integer         // carrier specified qps limit when
                                      // calling get data plan info API
  },
  "applicationIdentifiers": {
    "YouTube"    : carrier_app_id1,
    "Google Maps": carrier_app_id2,
  }
  "endPoints": {
   "ipPrefixes":
   [
    "ipPrefix1": "1.2.3.0/24",        // string with public IP prefix for
                                      // the operator's clients.
    "ipPrefix2": "4.5.6.0/24"
   ],
   "cpidEndPoints":
   [
    "cpidEndPoint1": "CPID_URL1",     // URL for client CPID query
    "cpidEndPoint2": "CPID_URL2",     // URL for client CPID query
   ],
   "dpaInfo": {
    "dpaEndPoints":
    [
     "dpaEndPoint1": "DPA_URL1",      // URL for DPA1
     "dpaEndPoint2": "DPA_URL2"       // URL for DPA2
    ],
    "dpaAccess": {
     "clientId": "google_api_client_id,
     "clientSecret": "google_api_client_secret",
     "accessScope":  "google_access_scope"
    }
   }
   "expiration": "2015-06-26T22:57:09+00:00", // ISO8601 Extended Time Format
  }
 }
}
```

If successful, this method returns a body describing the status of the server's
operation. The response body MUST have the following structure:

```
{
 "status": string
}
```

Google will cross-check the information provided (e.g., MCC/MNC, IP prefixes)
with public sources to detect any misconfigurations. Google will also provide
the list of possible application names.

### 6.2 Data Plan Metadata Registration Process {#6-2-data-plan-metadata-registration-process}

So far we have defined PMTCs as ENUM strings (e.g., MUSIC or GAMING). While such
strings may be descriptive to humans, applications cannot determine whether
their traffic will be included in a specific PMTC. For example, if a subscriber
has purchased a plan module with PMTC = MUSIC, Google Play Music alone cannot
determine whether the traffic it consumes will be rated against that plan module
without explicit communication and coordination with operators. To minimize this
overhead, GTAF introduces the following data plan metadata registration process
that binds app traffic to PMTCs.

Fundamentally apps do not have visibility into the semantics of PMTCs defined by
the operator. When querying GTAF for data plan status, an app should only
specify its name and the portion of app traffic that it is interested in.
Therefore, Google applications must define one or more "Application Traffic
Filter" strings representing relevant portions of app traffic to GTAF. For
example, YouTube may define "YOUTUBE_BROWSING" traffic filter string
corresponding to all browsing traffic. YouTube may also provide a list of SNIs
and IP prefixes that precisely classify YouTube app traffic that falls into
"YOUTUBE_BROWSING". Similarly, "YOUTUBE_VIDEO" represents all YouTube video
traffic.

Operators can now resolve this problem by defining mappings between PMTC strings
and the app traffic filter strings they correspond to. For example, an operator
can map the VIDEO PMTC to "YOUTUBE_VIDEO" filter string, which is optionally
accommodated by additional set of traffic filter strings: "Server Name
Indicators (SNIs): `*.googlevideo.com`".

To support this mapping, GTAF introduces the "`Carrier App Traffic Mapping`"
table (Table 1) in which operators can define their `PMTCs` with respect to
Google app traffic. This gives operators the flexibility to create new data
plans and manage their own `PMTC` name space with minimum coordination with
Google.

The "`Carrier App Traffic Mapping`" table requires the following information:

Table 1: Carrier App Traffic Mapping

<table>
  <tr>
   <td><strong><code>Column Name</code></strong>
   </td>
   <td><strong><code>Column Definition</code></strong>
   </td>
   <td><strong><code>Sample Value</code></strong>
   </td>
  </tr>
  <tr>
   <td><strong>Plan Module Traffic Category</strong>
<p>
<strong>(PMTC)</strong>
   </td>
   <td>Identifies a data plan module traffic category, as defined in <a
  href="#4-data-plan-terminology">Section 4</a>.
   </td>
   <td>"VIDEO"
   </td>
  </tr>
  <tr>
   <td><strong>Application Name</strong>
   </td>
   <td>A human-readable string describing a mobile app. The name might refer to
  multiple actual apps such as YouTube for Android, iOS, etc.
   </td>
   <td>"YouTube"
   </td>
  </tr>
  <tr>
   <td><strong>Carrier Application Identifier (CAID)</strong>
   </td>
   <td>A per-MNO unique ID used to refer to an application. This is the
  identifier that the application uses to request a CPID. It is the same as
  <code>Carrier Application Identifiers</code> used in the endpoint registration
  process above.
   </td>
   <td>"012xyAb"
    String mccMnc = ;
   </td>
  </tr>
  <tr>
   <td><strong>Application Package Name</strong>
   </td>
   <td>Standard package names for a mobile app in application stores (e.g.,
  Google Play or Apple Store), or the top level domain name for URL-based data
  plan/plan module.
   </td>
   <td>"com.google.android.youtube",
<p>
"com.google.ios.youtube"
   </td>
  </tr>
  <tr>
   <td><strong>Application Traffic Filter </strong>
   </td>
   <td>A list of traffic filter strings, pre-populated by Google app, or
  mutually agreed between Google app and carrier, used to identify the app
  traffic of a traffic category (PMTC), to GTAF.

In our current traffic ID solution these traffic filter strings can be lists of
   IP prefixes and SNIs but GTAF treats them as opaque strings.
   </td>
   <td>"sni:*.googlevideo.com"
<p>
"ip:74.125.224.0/24"
<p>
"YOUTUBE_VIDEO"
<p>
...
   </td>
  </tr>
  <tr>
   <td><strong>Application Traffic Descriptor </strong>
   </td>
   <td>A human-readable description of sub-application traffic. The application
   traffic descriptor can be used to describe one or more application traffic
   filters
   </td>
   <td>"YouTube Video Data"
   </td>
  </tr>
</table>

Cardinality mapping among various columns:

```
Plan Module Traffic Category (PMTC)     N:M Application Name
Application Name                        1:1 Carrier Application Identifier (CAID)
Carrier Application Identifier(CAID)    1:M Application Package Name
Application Traffic Descriptor          1:M Application Traffic Filter
PMTC + CAID                             1:1 App Traffic Descriptor
PMTC + CAID                             1:M Application Traffic Filter
```

To facilitate the registration process Google will pre-populate the Carrier App
Traffic Mapping table with entries for the relevant Google applications, such as
YouTube, as well as relevant traffic filter strings, such as YouTube browsing.

When an operator registers data plan modules that are applicable to a particular
Google app, they can look up the `application-traffic-filter`strings and fill up
remaining `PMTCs` (and `carrier application identifier`[^4]`)`field` `based on
current data plan offerings, since Google has already provided its `application
name`, `application package name, application traffic filter`and`application
traffic descriptor`. Operators MUST register data plan module metadata via a web
form or by issuing the following RESTful call:

```
PUT https://www.googleapis.com/ispportal/v0/datapackregistration
```

In the request body, supply data with the following structure:

```
{
 "planModuleRegistration": {         // plan module registration
  "pmtc": "VIDEO",                   // ENUM strings describing traffic type
  "appName": "YouTube",              // human-readable mobile app name
  "carrierAppId": "012abc",          // mutually agreed app id between app
                                     // service provider and carrier
  "appPackageNames": [               // standard app names in app store
    "com.google.android.youtube",
    "com.google.ios.youtube",
    "m.youtube.com"],
  "appTrafficFilters" : [            // list of traffic filter strings
    "*.youtube.com",
    "74.125.224.0/24",
    "YOUTUBE_BROWSING"
  ],
  "appTrafficDescriptor": "YouTube Video Data",
  }
}
```

Operators MUST update their app traffic mappings as PMTCs change. For example,
when the operator adds an application to a PMTC (e.g., a new game to the GAMING
PMTC) it should add the corresponding traffic filter string(s).

Table 2 shows how to represent the "ACME Blue" plan from[ Figure
1](#4-data-plan-terminology) in the carrier app traffic mapping table. Note that
"*" will match any application name (or application traffic filter string). For
example, the generic data plan will match application name "*" and application
traffic filter string "*". When YouTube app wants to know all plan modules that
are chargeable against video browsing, it would include the following query
parameters `app_name = "YouTube"` and `app_traffic_filter =
"YOUTUBE_BROWSING"`when calling GTAF data plan status API. GTAF then looks up
the carrier app traffic mapping table based on {`app_name`,
`app_traffic_filter}`, and finds out the matching `PMTCs` are `{VIDEO,
VIDEO_BROWSING, GENERIC}`in Table 2, implying YT browsing traffic falls into
`VIDEO/BROWSING/GENERIC` categories.

When the GTAF receives data plan status response from the DPA, it will filter
out any data plan module whose `PMTCs`do not contain any of the following:
`{VIDEO, VIDEO_BROWSING, GENERIC}.`

Table 2: Plan metadata registration for "ACME Blue" in carrier app traffic
mapping table.

<table>
  <tr>
   <td><strong>Plan Module Traffic Category (PMTC)</strong>
   </td>
   <td><strong>App Name</strong>
   </td>
   <td><strong>Carrier Application ID</strong>
   </td>
   <td><strong>Application Package</strong>
   </td>
   <td><strong>Application Traffic Descriptor</strong>
   </td>
   <td><strong>Application Traffic Filter</strong>
   </td>
  </tr>
  <tr>
   <td><code>'VIDEO'</code>
   </td>
   <td><code>'YouTube'</code>
   </td>
   <td><code>'yt123abc'</code>
   </td>
   <td><code>'com.google.android.youtube',</code>
<p>
<code>'com.google.ios.youtube'</code>
   </td>
   <td><code>'All YouTube Traffic'</code>
   </td>
   <td><code>*</code>
<p>
<code>'YOUTUBE_VIDEO'</code>
   </td>
  </tr>
  <tr>
   <td><code>'VIDEO_BROWSING'</code>
   </td>
   <td><code>'YouTube'</code>
   </td>
   <td><code>'yt123abc'</code>
   </td>
   <td><code>'com.google.android.youtube',</code>
<p>
<code>'com.google.ios.youtube'</code>
   </td>
   <td><code>'YouTube Browsing Traffic'</code>
   </td>
   <td><code>'*.youtube.com'</code>
<p>
<code>'YOUTUBE_BROWSING'</code>
   </td>
  </tr>
  <tr>
   <td><code>'GENERIC'</code>
   </td>
   <td><code>*</code>
   </td>
   <td><code>*</code>
   </td>
   <td><code>*</code>
   </td>
   <td><code>*</code>
   </td>
   <td><code>*</code>
   </td>
  </tr>
</table>

<!-- Footnotes themselves at the bottom. -->

## Notes

[^1]: including traffic from mobile browsers
[^2]: The VIDEO_OFFLINE PMTC means this plan is good for offline only (e.g.,
    really bad streaming QoE). It is independent of FlexTime window.
[^3]: Gray colored fields are common with dataPlanStatus.
[^4]: The carrier application identifier may be combined with endpoint
    registration process in Section 6.1.
