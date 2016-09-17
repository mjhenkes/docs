# CDS Services

## Discovery

```shell
curl "https://example.com/cds-services"
```

> The above command returns JSON structured like this:

```json
{
  "services": [
    {
      "hook": "patient-view",
      "name": "Static CDS Service Example",
      "description": "An example of a CDS service that returns a static set of cards",
      "id": "static-patient-greeter",
      "prefetch": {
        "patientToGreet": "Patient/{{Patient.id}}"
      }
    },
    {
      "hook": "medication-prescribe",
      "name": "Medication Echo CDS Service",
      "description": "An example of a CDS service that simply echos the medication being prescribed",
      "id": "medication-echo",
      "prefetch": {
        "patient": "Patient/{{Patient.id}}",
        "medications": "MedicationOrder?patient={{Patient.id}}"
      }
    }
  ]
}
```

Developers of CDS Services must provide a well-known endpoint allowing the EHR to discover all available CDS Services, including information such as the purpose of the CDS Service, when it should be invoked, and any data that is requested to be prefetched.

### HTTP Request

The discovery endpoint is always available at `{baseUrl}/cds-services`. For example, if the `baseUrl` is https://example.com, the EHR would invoke:

`GET https://example.com/cds-services`

### Response

The response to the discovery endpoint is an object containing a list of CDS Services.

Field | Description
----- | -----------
`services` | *array*. An array of **CDS Services**

Each CDS Service is described by the following attributes.

Field | Description
----- | -----------
`hook`| *string* or *url*. The hook this service should be invoked on. See [Hook Catalog](#hook-catalog)
`name`| *string*.  The name of this service
<nobr>`description`</nobr>| *string*. The description of this service
`id` | *string*. The {id} portion of the URL to this service which is available at<br />`{baseUrl}/cds-services/{id}`
`prefetch` | *object*. An object containing key/value pairs of FHIR queries to data that this service would like the EHR prefetch and provide on<br />each service call. The key is a *string* that describes the type<br />of data being requested and the value is a *string* representing<br />the FHIR query.<br />(todo: link to prefetching documentation)

### HTTP Status Codes

Code | Description
---- | -----------
`200 OK` | A successful response

## Calling a CDS Service

```
curl
  -X POST \
  -H 'Content-type: application/json' \
  --data @hook-details-see-example-below
  "https://example.com/cds-services/static-patient-greeter"
```

```json
{
   "hookInstance" : "d1577c69-dfbe-44ad-ba6d-3e05e953b2ea",
   "fhirServer" : "http://hooks.smarthealthit.org:9080",
   "hook" : "patient-view",
   "redirect" : "http://hooks2.smarthealthit.org/service-done.html",
   "user" : "Practitioner/example",
   "context" : [],
   "patient" : "1288992",
   "prefetch" : {
      "patientToGreet" : {
         "response" : {
            "status" : "200 OK"
         },
         "resource" : {
            "resourceType" : "Patient",
            "gender" : "male",
            "birthDate" : "1925-12-23",
            "id" : "1288992",
            "active" : true
         }
      }
   }
}
```

### HTTP Request

An EHR calls a CDS service by `POST`ing a JSON document to the service
endpoint, which can be constructed from the CDS Service base URL and an
individual serviec id as `{baseUrl}/cds-services/{service.id}`.  The CDS Hook
call includes a JSON POST body with the following input fields:

Field | Description
----- | -----------
`hook` |*string* or *URL*. The hook that triggered this CDS Service call<br />(todo: link to hook documentation)
<nobr>`hookInstance`</nobr> |*string*.  A UUID for this particular hook call (see more information below)
`fhirServer` |*URL*.  The base URL EHR's [FHIR](https://www.hl7.org/fhir/) server. The scheme should be `https`
`oauth` | *object*. The OAuth2 authorization providing access to the EHR's FHIR server (see more information below)
`redirect` |*URL*.  The URL an app link card should redirect to (see more information below)
`user` |*string*.  The FHIR resource type + id representing the current user.<br />The type is one of: [Practitioner](https://www.hl7.org/fhir/practitioner.html), [Patient](https://www.hl7.org/fhir/patient.html), or [RelatedPerson](https://www.hl7.org/fhir/relatedperson.html).<br />For example, `Practitioner/123`
`patient` |*string*.  The FHIR `Patient.id` of the current patient in context
`encounter` |*string*.  The FHIR `Encounter.id` of the current encounter in context
`context` |*object*.  Hook-specific contextual data that the CDS service will need.<br />For example, with the `medication-prescribe` hook this will include [MedicationOrder](https://www.hl7.org/fhir/medicationorder.html) being prescribed.
`prefetch` |*object*.  The FHIR data that was prefetched by the EHR (see more information below)

#### hookInstance

While working in the EHR, a user can perform
multiple activities in series or in parallel. For example, a clinician might prescribe
two drugs in a row; each prescription activity would be assigned a
unique `hookInstance`. The [[activity catalog|Activity]]
provides a description of what events should initiate and terminate
a given hook. This allows an external
service to associate requests with activity state, which is necessary to
support the following app-centric decision sequence, where
the steps are tied together by a common `hookInstance`:

  0. EHR invokes CDS hook
  1. CDS service returns *app link card*
  2. User clicks app link and interacts with app
  3. Flow returns to EHR, which re-invokes CDS hook
  4. CDS service returns *decision* with user's choice

Note: the `hookInstance` is globally unique and should contain enough entropy
to be un-guessable.

#### oauth

Security details allowing the CDS service to connect to the EHR's
FHIR server.  These fields allow the CDS service to access EHR data in a
context limited by the current user's privileges. `expires` expresses the
token lifetime as an integer number of seconds. `scope` represents the set of
scopes assigned to this token (see [SMART on FHIR
scopes](http://docs.smarthealthit.org/authorization/scopes-and-launch-context/)).
Finally, `token` is a bearer token to be presented with any API calls the CDS
service makes to the EHR, by including it in an Authorization header like:

    `Authorization: Bearer {{token}}`

#### redirect

This field is only used by services that will return an *app
link card*: when a user clicks the card's link to launch an app, it becomes
the app's job to send the user to *this `redirect` URL* upon completion of
user interaction. (*Design note*: this field is supplied up-front, as part of
the initial request, to avoid requiring the EHR to append any content to app
launch links. This helps support an important "degenerate" use case for app
link cards: pointing to static content. See below for details.)

#### prefetch

As a performance tweak, the EHR may pass along data according
to the service's [[Prefetch-Template]]. This helps provide the service with all
the data it needs to efficiently compute a set of recommendations. Each key matches a key described in the CDS Service Discovery document; and each value is a FHIR Bundle.entry indicating a response status and returned resource.

Note that in the absence of `prefetch`, an external service can always execute
FHIR REST API calls against the EHR server to obtain additional data ad-hoc.)

<aside class="notice">
You can see the <a href="http://editor.swagger.io/#/?import=https://raw.githubusercontent.com/cds-hooks/api/master/cds-hooks.yaml?token=AATHAQY8vqQ6dIZajRuuE55EWMBitTptks5XLMk6wA%3D%3D">complete data model in Swagger</a>.
</aside>

## CDS Service Response

> Example response

```json
{
  "cards": [
    {
      "summary": "Success Card",
      "indicator": "success",
      "detail": "This is an example success card.",
      "source": {
        "name": "Static CDS Service Example",
        "url": "https://example.com"
      },
      "links": [
        {
          "label": "Google",
          "url": "https://google.com"
        },
        {
          "label": "Github",
          "url": "https://github.com"
        }
      ]
    },
    {
      "summary": "Info card",
      "indicator": "info",
      "source": {
        "name": "Static CDS Service Example"
      }
    }
  ]
}
```

Field | Description
----- | -----------
`cards` |*array*. An array of **Cards**. Cards can provide a combination of information (for reading), suggested actions (to be applied if a user selects them), and links (to launch an app if the user selects them). The EHR decides how to display cards, but we recommend displaying suggestions using buttons, and links using underlined text.
<nobr>`decisions`</nobr> |*array*. An array of **Decisions**. A decision should only be generated after interacting with the user through an app link. Decisions are designed to convey any choices the user made in an app session.

represents a user's choice made while interactin with the CDS Provider's external app. The first call to a service should never include any `decisions`, since no user interaction has occurred yet.

Each **Card** is described by the following attributes.

Field | Description
----- | -----------
`summary` | *string*. one-sentence, <140-character summary message for display to the user inside of this card.
`detail` | *string*.  optional detailed information to display, represented in Markdown. (For non-urgent cards, the EHR may hide these details until the user clicks a link like "view more details...".) 
`indicator` | *string*.  urgency/importance of what this card conveys. Allowed values, in order of increasing urgency, are: `success`, `info`, `warning`, `hard-stop`. The EHR can use this field to help make UI display decisions such as sort order or coloring. The value `hard-stop` indicates that the workflow should not be allowed to proceed. 
`source` | *string*. grouping structure for a short, human-readable description (in `source.name`) of the source of the information displayed on this card, with an optional link (in `source.url`) where the user can learn more about the organization or data set that provided the information on this card. Note that `source.url` should not be used to supply a context-specific "drill-down" view of the information on this card. For that, use `link.url` instead.  
<nobr>`suggestions`</nobr> | *array* of **Suggestions**, which allow a service to suggest a set of changes in the context of the current activity (e.g.  changing the dose of the medication currently being prescribed, for the `medication-prescribe` activity)
`links` | *array* of **Links**, which allow a service to suggest a link to an app that the user might want to run for additional information or to help guide a decision.

Each **Suggestion** is described by the following attributes.

Field | Description
----- | -----------
`label` |  *string*. human-readable label to display for this suggestion (e.g. the EHR might render this as the text on a button tied to this suggestion).
`uuid` | *string*. unique identifier for this suggestion. For details see [Suggestion Tracking Analytics](#analytics)
<nobr>`create`</nobr> | *string*. new resource(s) that this suggestion applies within the current activity (e.g. for `medication-prescribe`, this holds the updated prescription as proposed by the suggestion).
`delete`  | *string*. id(s) of any resources to remove from the current activity (e.g. for the `order-review` activity, this would provide a way to remove orders from the pending list). In activities like `medication-prescribe` where only one "content" resource is ever relevant, this field may be omitted.

Each **Link** described by the following attributes.

Field | Description
----- | -----------
<nobr>`label`</nobr>| *string*. human-readable label to display for this link (e.g. the EHR might render this as the underlined text of a clickable link).
`url` | *URL*. URL to load (via `GET`, in a browser context) when a user clicks on this link. Note that this may be a "deep link" with context embedded in path segments, query parameters, or a hash. In general this URL should embed enough context for the app to determine the `hookInstance`, and `redirect` url upon downstream launch, because the EHR will simply use this url as-is, without appending any parameters at launch time.


Each **Decision** is described by the following attributes.

Field | Description
----- | -----------
`create` |*array* of *strings*. id(s) of new resource(s) that the EHR should create within the current activity (e.g. for `medication-prescribe`, this would be the updated prescription that a user had authored in an app session).
<nobr>`delete`</nobr> |*array* of *strings*. id(s) of any resources to remove from the current activity (e.g. for the `order-review` activity, this would provide a way to remove orders from the pending list). In activities like `medication-prescribe` where only one "content" resource is ever relevant, this field may be omitted.

# Analytics

Whenever a user clicks a button from a "suggestion" card, the EHR uses the
suggestion `uuid` to notify the CDS Service's analytics endpoint via a `POST`
with an empty body:

    `POST {baseUrl}/cds-services/{serviceId}/analytics/{uuid}`

If a suggestion has no `uuid`, the EHR does not send a notification.
