---
title: Patient Merge Interface
---

## Patient Merge Interface

Version 1.0

7 March 2017

The MITRE Corporation

### Introduction
### Purpose
This document describes the commands and objects used for the Patient Merging Service. This service provides a RESTful API using FHIR objects for requests and responses. This document will be updated periodically as additional functionality is added to the Patient Merging Service.

### Scope
This document describes the interaction between the Patient Merging Service and a FHIR server, with the aim to merge FHIR resources identified as duplicates into a new, updated FHIR resource.
The web service calls and data payloads are detailed below, as well as examples for resolving or aborting these merge operations. A later section documents convenience features for sharing a merge session.

### Assumptions
This document assumes FHIR Specification DSTU3, which is the current officially released version at the time of this writing.

### Technical Approach
The Patient Merging Service is invoked by a client providing two FHIR URLs for Patient resources, corresponding to patient data assumed to be similar enough to warrant a desire to merge into a single resource. This invocation can be referred to as a "merge session" for the remainder of this document.
If the records can be successfully be auto merged by the service, the service creates a new FHIR resource and sends the resulting Bundle (referred to as the **target** Bundle) back to the client in the response.
If the records need intervention to choose the correct merge result, an OperationOutcome is returned to the client, with IDs to the conflicting data values. The user enters or chooses the correct data, resubmits the request to the service (via the **/resolve** endpoint), and the validation continues to resolve the new object.
The resolving of conflicting data is designed to take place on an incremental basis if the client desires (e.g. resolve conflicting medication dosages, resubmit, then resolve a conflicting encounter dates, resubmit, etc.).
The client can also choose to abort the merge session, which will stop the creation of the new FHIR resource, deleting the in-progress, new resource as well.
A merge session can also be transmitted to another user (e.g. a clinician) for consultation on the appropriate value that should be used to resolve a merge.


### Use Cases
The first use case is a user interface program that offers fields and actions for viewing and updating merge data. The second use case is an outside party receiving the URL for a merge session, wherein that party can then modify and resubmit the correct data, using the same endpoints.

### Message Definitions
The ptmerge service is exposed as a RESTful API. All communication with the ptmerge service is done through the following API routes:

* [Start a New Merge](#post-merge)
* [Resolve a Merge Conflict](#post-mergemerge_idresolveconflict_id)
* [Abort or Delete a Merge](#post-mergemerge_idabort)
* [Get Info on All Merge Sessions](#get-merge)
* [Get Info on a Specific Merge Session](#get-mergemerge_id)
* [Get A Merge's Target Bundle](#get-mergemerge_idtarget)
* [Get A Merge's Outstanding Conflicts](#get-mergemerge_idconflicts)
* [Get A Merge's Resolved Conflicts](#get-mergemerge_idresolved)

#### `POST /merge`

Initiates a new merge session, when, if successful, returns a merged bundle. Fully qualified URLs to two FHIR bundles must be provided as query parameters:

* `source1` - The fully qualified URL referencing source bundle 1.
* `source2` - The fully qualified URL referencing source bundle 2.

There are 2 simple ways to construct a path to source bundles:

`GET [base]/Bundle/:id`

`GET [base]/Patient/:id/$everything`

And one way that offers more control over the content in the source bundle:

```
GET [base]/Patient?_id=:id&_include=<paths_to_include>&_revinclude=<paths_to_include>
```

Where the paths to `_include` and `_revinclude` for a given resource are described in the FHIR server's [CapabilityStatement](https://github.com/synthetichealth/gofhir/blob/stu3_jan2017/conformance/capability_statement.json). In practice, `$everything` is actually the union of `_include=*&_revinclude=*`

During the merge the two source bundles **will not, and should not be modified in any way**.

#### Request

```
POST /merge?source1=http://fhir.example.com/Bundle/58939a3188a94d1a28634257&source2=http://fhir.example.com/Bundle/58939a3188a94d1a28634206
```

The POST body should be empty. Any content in the body will be ignored.

#### Responses

| Status | Description  |
|:-------|:-------------|
| `200` | **The merge had no conflicts and succeeded immediately**. The response body contains the complete, merged bundle. |
| `201` | **The merge has one or more outstanding conflicts**. The response body contains a bundle of OperationOutcomes detailing the merge conflicts. |
| `500` | **An unexpected error occurred**. That's all we know. The current state of the merge session is unknown. |

##### `200 OK`: Successful Merge

If a merge succeeds immediately, ptmerge responds with the complete, merged bundle in JSON format:

```json
{
  "resourceType": "Bundle",
  "id": "58939a3188a94d1a28634257",
  "type": "collection",
  "entry": [
    {
      "resource": {
        "resourceType": "Patient",
        "id": "58939a3188a94d1a28634257",
        "name": [{
          "family": "Abbott",
      	  "given": ["Clint"]
        }],
        "gender": "male",
        "birthDate": "1950-09-02"
      }
    }
  ]
}
```

##### `201 Created`: Merge With Conflicts

If a merge has conflicts, ptmerge responds with a bundle of one or more OperationOutcome resources. Each OperationOutcome describes a single merge conflict. A new **merge session** is started and the `merge_id` is returned in the `Location` header. A new "target" bundle is created on the host FHIR server and is populated with merged resources as the merge session progresses. The OperationOutcomes in the response are also created on the host FHIR server.

```json
{
  "id": "58939a3188a94d1a28634257",
  "resourceType": "Bundle",
  "type": "transaction-response",
  "entry": [
    {
      "resource": {
        "resourceType": "OperationOutcome",
        "id": "58b5856c97bba9a4096d9916",
        "issue": [
          {
            "severity": "information",
            "code": "conflict",
            "diagnostics": "Patient:58b5856b97bba9a40db1a52a",
            "location": [
              "maritalStatus.coding[0].display",
              "maritalStatus.coding[0].code"
            ]
          }
        ]
      },
      "response": {
        "status": "201"
      }
    }
  ]
}
```

The `diagnostics` field captures information about the target resource for each conflict in the target bundle. This field indicates the resource type of the target (e.g. `Patient`) and its ID (e.g. `58b5856b97bba9a40db1a52a`), separated by a colon.

### `POST /merge/:merge_id/resolve/:conflict_id`

Attempts to resolve a merge conflict. The request body _must_ contain the complete resource that resolves the conflict. To avoid ambiguity a valid request **always resolves the merge conflict.**

`:merge_id` - The ID referring to the current merge session.

`:conflict_id` - The ID referring to the merge conflict that is currently being resolved. This matches an OperationOutcome ID.

#### Request

```
POST /merge/58939a3188a94d1a28634257/resolve/58939a3188a94d1a28634110
```

The POST body must contain the resource that resolves the conflict, in FHIR JSON format. For example:

```json
"resource": {
  "resourceType": "Encounter",
  "id": "58939a3188a94d1a28634257",
  "status": "finished",
  "type": [{
    "coding": [{
      "system": "http://www.ama-assn.org/go/cpt",
      "code": "99201"
    }],
    "text": "Office Visit"
  }],
  "patient": {
    "reference": "5a909a318da94d1h28i34236"
  },
  "period": {
    "start": "2012-09-20T08:00:00-05:00"
  }
}
```

#### Responses

| Status | Description  |
|:-------|:-------------|
| `200` | **The conflict resolution request succeeded**. This does not necessarily mean that the conflict was resolved. The response body contains a bundle detailing the outcome of the conflict resolution. |
| `400` | **Bad request**. This may occur if the specific merge session or merge conflict was already resolved. |
| `404` | **Not found**. The specified merge session or merge conflict does not exist. |
| `500` | **An unexpected error occurred**. That's all we know. The current state of the merge session is unknown. |

##### `200 OK`: The Resolution Request Succeeded

There are two possible outcomes in this scenario:

1. **The last merge conflict in this session was resolved**. The response body contains the complete, merged bundle as FHIR JSON.

2. **The conflict was resolved, but additional conflicts still exist**. The response body contains the current set of unresolved merge conflicts, as a bundle of OperationOutcomes.


### `POST /merge/:merge_id/abort`

Terminates an in-progress merge session. **This operation cannot be undone**.

`:merge_id` - The ID referring to the current merge session.

An abort request also:

1. Deletes all FHIR resources related to the merge.
2. Deletes any record of the merge session.

#### Request

```
POST /merge/5a909a318da94d1h28i34236/abort
```

The POST body should be empty. Any content in the body will be ignored.

#### Responses

| Status | Description  |
|:-------|:-------------|
| `204` | **The merge was successfully aborted**. The target bundle and all OperationOutcomes that were part of this merge session were also deleted. |
| `404` | **Not found**. The specified merge session does not exist. |
| `500` | **An unexpected error occurred**. That's all we know. The current state of the merge session is unknown. |

### `GET /merge`

Get the metadata for all merge sessions that ptmerge has processed.

#### Request

```
GET /merge
```

#### Responses

**This route returns custom JSON, not FHIR JSON.**

| Status | Description  |
|:-------|:-------------|
| `200` | **The metadata request succeeded**. The response body contains the complete metadata for all known merge sessons. |
| `500` | **An unexpected error occurred**. That's all we know. |

##### `200 OK`: Valid Request

If the request is valid, ptmerge responds with the JSON metadata for all known merge sessions. For example:

```json
{
  "timestamp": "2017-02-28T09:31:57.280949164-05:00",
  "merges": [
    {
      "id": "58b48f7497bba924465bc141",
      "targetBundle": "http://localhost:3001/Bundle/58b48f7497bba924cd97de03",
      "conflicts": {
        "58b48f7497bba924cd97de04": {
          "operationOutcome": "http://localhost:3001/OperationOutcome/58b48f7497bba924cd97de04",
          "targetResource": {
            "id": "58b48f7497bba924465bc137",
            "type": "Encounter"
          },
          "resolved": false
        },
        "58b48f7497bba924cd97de05": {
          "operationOutcome": "http://localhost:3001/OperationOutcome/58b48f7497bba924cd97de05",
          "targetResource": {
            "id": "58b48f7497bba924465bc13b",
            "type": "Patient"
          },
          "resolved": false
        }
      },
      "completed": false
    },
    {
      "id": "58b5820d97bba9a40db1a51e",
      "targetBundle": "http://localhost:3001/Bundle/58b5820d97bba9a4096d9911",
      "conflicts": {
        "58b5820d97bba9a4096d9912": {
          "operationOutcome": "http://localhost:3001/OperationOutcome/58b5820d97bba9a4096d9912",
          "targetResource": {
            "id": "58b5820d97bba9a40db1a513",
            "type": "Patient"
          },
          "resolved": false
        },
        "58b5820d97bba9a4096d9913": {
          "operationOutcome": "http://localhost:3001/OperationOutcome/58b5820d97bba9a4096d9913",
          "targetResource": {
            "id": "58b5820d97bba9a40db1a517",
            "type": "Encounter"
          },
          "resolved": true
        }
      },
      "completed": false
    }
  ]
}
```

In this example there are 2 known merge sessions:

1. The first is an in-progress merge session with 2 conflicts, both outstanding.

2. The second is a merge session with 2 conflicts, one resolved.

### `GET /merge/:merge_id`

Get the metadata for a specific merge session.

`:merge_id` - The ID referring to the specific merge session.

#### Request

```
GET /merge/58939a3188a94d1a28634257
```

#### Responses

| Status | Description  |
|:-------|:-------------|
| `200` | **The metadata request succeeded**. The response body contains the metadata for one, specific merge. |
| `404` | **Not found**. The specified merge session does not exist. |
| `500` | **An unexpected error occurred**. That's all we know. |

##### `200 OK`: Valid Request

If the request is valid, ptmerge responds with the JSON metadata for the merge session. For example:

```json
{
  "timestamp": "2017-02-28T10:27:50.531954239-05:00",
  "merge": {
    "id": "58b5856c97bba9a40db1a52f",
    "targetBundle": "http://localhost:3001/Bundle/58b5856b97bba9a4096d9914",
    "conflicts": {
      "58b5856c97bba9a4096d9915": {
        "operationOutcome": "http://localhost:3001/OperationOutcome/58b5856c97bba9a4096d9915",
        "targetResource": {
          "id": "58b5856b97bba9a40db1a526",
          "type": "Encounter"
        },
        "resolved": true
      },
      "58b5856c97bba9a4096d9916": {
        "operationOutcome": "http://localhost:3001/OperationOutcome/58b5856c97bba9a4096d9916",
        "targetResource": {
          "id": "58b5856b97bba9a40db1a52a",
          "type": "Patient"
        },
        "resolved": true
      }
    },
    "completed": false
  }
}
```

### `GET /merge/:merge_id/target`

Get a merge session's target bundle.

`:merge_id` - The ID referring to the specific merge session.

#### Request

```
GET /merge/58939a3188a94d1a28634257/target
```

#### Responses

| Status | Description  |
|:-------|:-------------|
| `200` | **The request succeeded**. The response body contains the up-to-date target bundle. |
| `404` | **Not found**. The specified merge session does not exist. |
| `500` | **An unexpected error occurred**. That's all we know. |

Note that the bundle returned in a successful response **may be partially complete**. This depends entirely on the current state of the merge session.

### `GET /merge/:merge_id/conflicts`

Get a merge session's outstanding merge conflicts.

`:merge_id` - The ID referring to the specific merge session.

#### Request

```
GET /merge/58939a3188a94d1a28634257/conflicts
```

#### Responses

| Status | Description  |
|:-------|:-------------|
| `200` | **The request succeeded**. The response body contains a bundle of OperationOutcomes detailing the merge session's outstanding merge conflicts. If the merge is complete or no more conflicts exist, this may be an empty bundle.|
| `404` | **Not found**. The specified merge session does not exist. |
| `500` | **An unexpected error occurred**. That's all we know. |

### `GET /merge/:merge_id/resolved`

Get a merge session's resolved merge conflicts.

`:merge_id` - The ID referring to the specific merge session.

#### Request

```
GET /merge/58939a3188a94d1a28634257/resolved
```

#### Responses

| Status | Description  |
|:-------|:-------------|
| `200` | **The request succeeded**. The response body contains a bundle of OperationOutcomes detailing the merge session's resolved merge conflicts. If no merge conflicts were resolved yet, this may be an empty bundle. |
| `404` | **Not found**. The specified merge session does not exist. |
| `500` | **An unexpected error occurred**. That's all we know. |
