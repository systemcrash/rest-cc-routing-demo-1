# REST CC (Call Control) DB

Use REST Call Control capabilities of your SBC together with a RESTful database to free your SBC of business logic. This example shows how to select a destination PBX IP from the RESTful JSON database, based on the Request-URI (i.e. the To: or B number).

Keywords: REST, JSON, cURL, SBC

Required Firmware: v6.3.1

## Description

You want to choose a destination IP based on the callee, or the Request-URI/To header (B number). 


## Detailed Overview 

Your SBC (remote to this repo and which has Internet access) captures a Request-URI from an ingress INVITE, and sends it in a concurrent cURL request to the endpoint hosting this JSON database. In this example database there are three fields for each entry:

- `PBXIP` - holds the fake destination domain - this will be the outbound Request-URI domain portion
- `RURI` - holds the number to match from the `Request-URI`
- `Comment` 


So for `RURI = 34915552345`:

| Routing      | 0 |
|---------|---|
| RURI    |34915552345|
| PBXIP   |fakedomain1|
| Comment |PBX A (Madrid)|


When the cURL request returns, the values in this example are supplied to regular expressions which direct egress requests based on these provided values. In effect, we rewrite our destination domain, based on the Request-URI, stored in a remote database.

[Click here]( https://my-json-server.typicode.com/systemcrash/rest-cc-routing-demo-1/routing?RURI=34915552345 ) to see these example values for `RURI = 34915552345` returned by the RESTful database.

Then, use a DNS Override to translate `fakedomainX` to an IP.

## Resources and dependencies

Firmware >= 6.3.1

[My JSON Server](https://my-json-server.typicode.com/) which describes itself as a `Fake Online REST server for teams`. Read more about it to understand how this demonstration works. For this demo, My JSON Server uses the `db.json` file in this repository.

On your Ingate SBC, under:
- __SIP Services, Basic Settings__: You must have a SIP TCP port configured (only UDP or TLS is not sufficient). It does not need to be `Active`.

Not required but a highly recommended guide if you do not understand Regular Expressions (regexp):

[Regular Expressions for Regular Folk](https://github.com/shreyasminocha/regex-for-regular-folk)


Note: hosting your own instance is a good option. TLS up to you. 

## Worked Example: Rewrite the destination domain

Vicent wants to divide regional Spanish numbers between two different PBX IPs. One PBX is not enough to handle the amount of extensions and phone numbers he has. But the SBC is not configured yet. So it's time to configure the SBC to use the RESTful database for destination selection. 

---
Under the Call Control tab, add a REST API Server row:

| ... | ID  | Prefix | Suffix | ... |
|-----|------|-----|--------|-----|
|     | `2`  |`https://my-json-server.typicode.com/your-github-repo/rest-cc-routing-demo-1/routing?RURI=`|        |     |

Whereby
|Path|Description|
|---|---|
|`your-github-repo/` |your github account/org name|
|`rest-cc-routing-demo-1/`|this repo path|


You can even try with this repo as your prefix - everything is ready to use: `https://my-json-server.typicode.com/systemcrash/rest-cc-routing-demo-1/routing?RURI=`


Note: Later in the dial plan, the shortcut `$curl2` refers to the Server ID `2` above. To use `$curl3`, add a row with ID `3`, and so on.

Note: to adjust cURL timeout on v6.2.2 cumulatively patched, add (basic,advanced,options) advanced option name (sip.cc-timeout) and a value. Firmware >= 6.3.x has a dedicated field for this under REST CC.

---

In the dial plan, add a new:
* `Matching R-URI` row which captures a number from the Request-URI:

| ... | Name | ... | Regexp | ... |
|-----|------|-----|--------|-----|
|     | `Any` |     | `sip:\+?(.*)@.*` |     |

Note: Capture group `$1` or `$r1` is `(.*)`.

* `Forward To` row which invokes our cURL expression:

| ... | Name | ... | Regexp | Trunk |...|
|-----|------|-----|--------|-----|---|
|  |`RESTCCrouting`| | `sip:$1@$curl2($r1_XPATH//PBXIP/text())`|-| |


Note: `$r1` expresses the first capture group from our ingress Request-URI. In this example `$r1`, supplies the Request-URI to the `$curl2(...)` expression.

* `Dial Plan` row: 

| ... | From | Request URI | Action | Forward To | ... |
|-----|------|-----|--------|-----|---|
|     | ...  | ... | ... | ... |
|     | ...  | `Any` | Forward | `RESTCCrouting` |


---

Under SIP Traffic -> Routing -> DNS Overrides, add these entries:


| ... | Domain      | IP            | Transport | ... | Modify R-URI |
|-----|-------------|---------------|-----------|-----|--------------|
|     | fakedomain1 | 172.31.254.10 | UDP       | ... | Yes |
|     | fakedomain2 | 172.31.254.11 | UDP       | ... | Yes |
|     | fakedomain3 | 172.31.254.12 | UDP       | ... | Yes |
|     | fakedomain4 | 172.31.254.13 | UDP       | ... | Yes |

Adjust the values as appropriate for your environment.

SIP Routing Order should read:

* 1 DNS Override
* 2 Local Registrar
* 3 Dial Plan


Our concurrent cURL request becomes [`https://my-json-server.typicode.com/systemcrash/rest-cc-routing-demo-1/routing?RURI=34915552345`]( https://my-json-server.typicode.com/systemcrash/rest-cc-routing-demo-1/routing?RURI=34915552345 ), to which the response comes 
```json
[
  {
    "RURI": 34915552345,
    "PBXIP": "fakedomain1",
    "comment": "PBX A (Madrid)"
  }
]
```


Note: 

The Call Control function converts received JSON to XML internally, whereupon XPATH expressions are available. Our `$curl2` expression has `XPATH//PBXIP/text()`. This takes the `PBXIP` field text (which is a *text* field in the JSON above, but everything gets converted to text anyway). 


With everything configured and applied now, here is an ingress SIP INVITE to +34915552345:

```http
INVITE sip:+34915552345@192.0.2.5 SIP/2.0
Via: SIP/2.0/UDP 192.0.2.254:5060;branch=z9hG4bK-909121779;received=192.0.2.254;rport=5060
Content-Length: 0
From: "Vicen" <sip:90501@192.0.2.254>;tag=633
Accept:
User-Agent: Vicens-phone
To: "Madrid" <sip:+34915552345@192.0.2.5>
Contact: sip:90501@192.0.2.254:5060
CSeq: 1 INVITE
Call-ID: 355593493830278013560573
Max-Forwards: 70
```

The Request-URI contains the user portion `+34915552345`. An INVITE that egresses to our PBX might look like:

```http
INVITE sip:+34915552345@172.31.254.10 SIP/2.0
Via: SIP/2.0/UDP 172.31.254.1:5060;branch=z9hG4bK-909121779
Content-Length: 0
From: "John" <sip:90501@172.31.254.1>;tag=633
User-Agent: Franken-SBC
To: "USA" <sip:+34915552345@fakedomain1>
Contact: sip:zyx123@172.31.254.1
CSeq: 1 INVITE
Call-ID: 1
Max-Forwards: 70
```


## Code and Design (how to reproduce this JSON DB file)

Draw up your javascript DB at the following location

[https://beta5.objgen.com/json/local/design](https://beta5.objgen.com/json/local/design)

Clarification of the below JSON object design:

```json

routing[]  <-- An array [], named 'routing' which will hold all of the routing.
  RURI n = 34915552345 <-- Number, the caller ID from the Request-URI
  PBXIP = fakedomain1 <-- String, the fake destination domain
  comment = PBX A (Madrid)

```

Copy the below example JSON object code and paste at the above URL to produce formatted JSON output which will make our database file `db.json`, ready for my-json-server. 

```json
routing[]
  RURI n = 34915552345
  PBXIP = fakedomain1
  comment = PBX A (Madrid)

routing[]
  RURI n = 34965552345
  PBXIP = fakedomain2
  comment = PBX B (Alicante)

routing[]
  RURI n = 349575552345
  PBXIP = fakedomain3
  comment = PBX C (Cordoba)

routing[]
  RURI n = 34955552345
  PBXIP = fakedomain4
  comment = PBX D (Malaga/Sevilla)

profile
  name = typicode
```

---

The above JSON object model code should produce the following JSON:
```json

{
  "routing": [
    {
      "RURI": 34915552345,
      "PBXIP": "fakedomain1",
      "comment": "PBX A (Madrid)"
    },
    {
      "RURI": 34965552345,
      "PBXIP": "fakedomain2",
      "comment": "PBX B (Alicante)"
    },
    {
      "RURI": 349575552345,
      "PBXIP": "fakedomain3",
      "comment": "PBX C (Cordoba)"
    },
    {
      "RURI": 34955552345,
      "PBXIP": "fakedomain4",
      "comment": "PBX D (Malaga/Sevilla)"
    }
  ],
  "profile": {
    "name": "typicode"
  }
}

```
---


