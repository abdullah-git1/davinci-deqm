
### Introduction

Clinical Quality Measures (CQMs) are a common tool used throughout healthcare to help evaluate and understand the impact and quality of the care being provided to an individual or population. The intent of [data of interest] is the source data needed to calculate a quality measure, as specified by the data requirements of the measure. For example, for a colorectal cancer screening measure, the data of interest is the set of conditions, procedures, and observations related to determining whether a patient is in the initial population, denominator, and numerator of the quality measure. To effectively evaluate quality measures in such an environment requires timely exchange of the relevant data.

Transactions between Consumers (organizations that want to evaluate quality measures) and Producers (organizations that deliver care to patients) are triggered by use case specific clinical or administrative events such as the completion of a Medication Reconciliation or a request from a Payer for the attestation information. Note that although triggering is implementation specific and out of scope for this IG,  there are a variety of potential triggering points for reporting events within clinical systems.  These include:

* [Infobutton] event listing
* [eCR] Event Types
* [CDS Hooks] Hook definitions
* [CPG-on-FHIR] common processes

and there will be a need to convey triggering information in a computable way to EHRs and other clinical systems.

This Implementation Guide (IG) describes two methods of exchanging data quality information using a set of [FHIR operations] that provide the framework to exchange CQM data:

1. CQM data may be submitted to the Consumer by the Producer using the [Submit Data scenario](#submit-data)
1. CQM data may be requested from the Producer by the Consumer using the [Collect Data scenario](#collect-data)

Note that FHIR operations allow the implementation to be viewed as a 'black box' free to decide how to satisfy the query - "give me the data of interest for a measure" - without requiring generic FHIR search functionality.

  This project recognizes the impact of the [Argonaut Clinical Data Subscriptions] project which is working on event based subscriptions and major revisions to the Subscription resource for FHIR R5. In a future version this guide, a subscription based exchange <!-- in which the Consumer may subscribe to a Producer's Subscription service to be notified when the CQM data is available --> is planned and will align with the outcomes of the Argonaut project.
  {:.stu-note}

### Default Profiles

The following resources are used in all data exchange transactions:


| Resource Type | Profile Name                             | Link to Profile                            |
|---------------|------------------------------------------|--------------------------------------------|
| Library       | CRMI Shareable Library                   | [CRMI Shareable Library]                   |
| Measure       | CRMI Shareable Measure                   | [CRMI Shareable Measure]                   |
| MeasureReport | DEQM Data Exchange MeasureReport Profile | [DEQM Data Exchange MeasureReport Profile] |
| Organization  | QI Core Organization Profile             | [QI Core Organization]                     |
| Patient       | QI Core Patient Profile                  | [QI Core Patient]                          |
| Practitioner  | QI Core Practitioner Profile             | [QI Core Practitioner]                     |


Depending on the specific Measure, various DEQM and QI Core Profiles are also used in addition to the profiles listed above

#### Graph of DEQM Resources
{:.no_toc}

The DEQM resources form a network through their relationships with each other - either through a direct reference to another resource or through a chain of intermediate references. These groups of resources are referred to as resources graphs.  The DEQM data exchange resource graph is shown in Figure 2-1, below. The resource graph shows data for a single measure for a single patient, but as described below and in [DEQM Operations Bundles Organized by Subject](guidance.html#deqm-operation-bundles-organized-by-subject), the DEQM data exchange operations are designed to allow for multiple measures across multiple patients.

{% include img.html img="measure-resource-graph.jpg" caption="Figure 2-1 DEQM Resource Graph" %}


### Submit Data
{: #submit-data .toc}

{:.highlight-note}
The [$submit-data] operation allows a Producer to submit data of interest for one or more measures, and for one or more subjects, within the specified [submission period].  The operation MAY be repeated during the submission period as additional data relevant to the quality measure becomes available.  The Producer submits the data either as  [incremental] or [snapshot] updates. These update methods are described in detail [below](#submit-updates).

 Alternatively, data may be submitted in bulk with the [$bulk-submit-data](OperationDefinition-bulk-submit-data.html) operation. This supports both synchronous and asynchronous messaging as described in the operation definition.

{% include img.html img="submit-data-step.jpg" caption = "Figure 2-2 Submit Data Steps" %}

#### Gather Data Requirements from Consumer
{:.no_toc}

To support the Submit Data operation, an implementation needs to know specifically what data are required to provide as the payload for the operation.  As described in the [Background] section of this guide, the profiles used in measuring and reporting CQMs are developed through a multi-stakeholder consensus-based process and are made available to the Producer.  The Producer is able to query for profiles needed for reporting a given measure and the criteria for the sending of the data.  This can be done manually by reviewing the measure definition or computationally by invoking the *Data Requirements* operation on a Consumer's measure instance endpoint as described below. These profiles are subsequently referenced in the `MeasureReport.evaluatedResources` element when submitting the measure data to the Consumer.

Note that because the data exchange scenarios described are intended to support exchange throughout a measurement period, the versions of measure specifications may change during the measurement period. Care should be taken to ensure the appropriate versioning of measure specifications and the impact of those changes on data exchanged using these methods

{% include img-narrow.html img="data-requirement.jpg" caption="Figure 2-3 Data Requirements Operation" %}

##### APIs
{:.no_toc}

In addition to the resources listed above, the following artifacts are used in this transaction:

1. Data Requirements: [$data-requirements] operation

##### Usage
{:.no_toc}

 The required data for each Measure is discovered by invoking the *Data Requirements* operation on the Consumer's `Measure/[measure-id]` endpoint.  Using either the `GET` or `POST` Syntax, the operation can be invoked as follows:

`GET|[base]/Measure/[measure-id]/$data-requirements?periodStart={periodStart}&periodEnd={periodEnd}`

`POST|[base]/Measure/[measure-id]/$data-requirements`

Note the use of the `periodStart` and `periodEnd` parameters supports description of data requirements that filter based on the measurement period. For example, a measure may require data for a certain procedure within the last three years, and the data requirements returned will reflect that period.

{% include error-note.md transaction = 'Data Collection' %}

{% include examplebutton.html example="data-requirements-example" b_title = "Click Here To See Example Data Requirements operation" %}

For another example see the [COL Data Requirements Operation] example.

#### Submit Data Operation
{:.no_toc}


Once the Producer understands the data requirements, they will use the *Submit Data* operation to submit bundles of MeasureReports and the referenced resources as discovered by the *Data Requirements* operation to the Consumer. There is no expectation that the submitted data represents all the data of interest, only that all the data submitted is relevant to the calculation of the measure for a particular subject or population. The Consumer simply accepts the submitted data and there is no expectation that the Consumer will actually evaluate the quality measure in response to every Submit Data. In addition, the Submit Data operation does not provide for analytics or feedback on the submitted data.

{% include img-narrow.html img="submit-data.jpg" caption="Figure 2-4 Submit data Operation" %}

<div class="highlight-note" markdown="1">

##### Incremental and Snapshot Updates
{:.no_toc #submit-updates}

When the Producer submits updates to the measure data within the data submission period, the Producer can use [snapshot] or [incremental] updates for submitting data based on Producer and Consumer agreement.  Note that neither method is preferred or a default and has to be agreed upon out of band.

Examples of patient ‘events’ that could trigger the submission of an update:

- A visit, including telemedicine, to a physician’s office.
- Being discharged from a hospital.

**Discovery:**

  - A CapabilityStatement is retrieved from a FHIR endpoint:

  `GET|[base]/metadata`

  - The Consumer (server) **SHALL** advertise whether it supports snapshot and/or incremental data exchange via its CapabilityStatement using the [DEQM Submit Data Update Type Extension].  The extension is added to the `CapabilityStatement.rest.resource.operation` element for the Submit Data operation and the code is valued `incremental` or `snapshot`, as appropriate.  The snippet shown below would be used for supporting both `incremental` and `snapshot` data exchanges:

     {% include CapabilityStatement-updatetype-snippet.md %}

   (For a complete example showing support for only the `incremental` data exchange, see the [Consumer Server CapabilityStatement] )

   - It is the responsibility of the Producer (client) to discover whether snapshot, incremental or both data exchange is supported by the inspection of the Consumer’s (server) CapabilityStatement.

   - The Producer (client) **SHALL** populate the required [DEQM Submit Data Update Type Extension] on the [DEQM Data Exchange MeasureReport Profile] to indicate whether the payload is a snapshot or incremental update for both the initial transaction and subsequent updates.

  - The Consumer (server) **SHALL** reject the submit data payload if there is mismatch between the update type stated in the data exchange MeasureReport submitted by the Producer (client) and the capabilities supported by the Consumer (server)  by returning a `400 Bad Request` http error code. An OperationOutcome **SHALL** be returned stating that the [snapshot/incremental] update is not supported as shown in the following example:

    {% include snapshotincremental-notsupported-oo.md %}

**Incremental Update Requirements and Expectations:**

  - For incremental data exchange, stable logical (resource) ids and meta.source elements are required for *ALL* transacted resources across *ALL* transactions.

    - For example  `MeasureReport.id` + `MeasureReport.meta.source`, `Patient.id` + `Patient.meta.souce`, etc ... must be the same for all data exchange interactions for a patient and measure during the submission period.

  - Note that resource versions can be accessed using the FHIR RESTful history transaction

**Snapshot Update Requirements and Expectations:**

  - Snapshot data exchange overwrites previous data ( in other words, the Producer resubmit all data for multiple patients even for a single error).

  - Snapshot data exchange is appropriate when the Consumer system is stateless (doesn’t persist data)

    - Otherwise the Consumer may have to save previous snapshots

  - Snapshot data exchange is appropriate when the Producer can’t support stable logical (resource) ids

</div>

##### APIs
{:.no_toc}

In addition to the resources listed above, the following artifacts are used in this transaction:

1. Submit Data operation: [$submit-data]
1. Various DEQM and QI Core Profiles depending on the specific Measure

##### Usage
{:.no_toc}

Using the `POST` Syntax, the operation can be invoked by the Producer:

`POST|[base]/Measure/[measure-id]/$submit-data`

{% include error-note.md transaction = 'Submit Data' %}

{% include examplebutton.html example="submit-data-example" b_title = "Click Here To See Example Submit Data Operation (edited for brevity)" %}

For a complete un-edited example see the [MRP Submit Data Operation] and [COL Submit Data Operation] examples.

### Collect Data
{: #collect-data}

In this scenario, the Consumer initiates a [$collect-data] operation to gather any available CQM data for a particular measure from the Producer.  In response to the operation, the Producer returns a MeasureReport containing data relevant to the Measure. The Producer gathers the data requirements as [described](#gather-data-requirements-from-consumer) above in the Submit Data scenario. Like the Submit Data scenario, there is no expectation that the data returned represents all the data required to evaluate the quality measure only that all the data submitted is relevant to the calculation of the measure for a particular subject or population.  Unlike the Submit Data interaction, the exchange is typically incremental as detailed [below](#).

{% include img.html  img="collect-data-steps.jpg" caption = "Figure 2-5 Collect Data Steps"%}

#### Collect Data Operation
{:.no_toc}


The Consumer uses a Collect Data operation to request any available relevant data for the evaluation of measures from a Producer. Unlike the Submit Data interaction, the collect data exchange is typically incremental. This would typically be done on a periodic basis to support incremental collection of quality data. The `lastReceivedOn` parameter can be used to indicate when the last Collect Data operation was performed, allowing the Producer to limit the response to only data that has been entered or changed since the last received on date.

{% include img-narrow.html img="collect-data.jpg" caption="Figure 2-6 Collect data Operation" %}

##### Incremental and Snapshot Updates
{:.no_toc #collect-updates}

- Unlike the Submit Data interaction, there is no need for out of band discovery.
- The Consumer uses the Collect Data operation’s `lastReceivedOn` parameter for incremental data exchange - if the  parameter present, it is an incremental update and snapshot if not.
- The same Snapshot and Incremental Requirements and Expectations described above for the Submit Data transaction apply for Collect Data.
- If the Producer cannot support the lastReceivedOn parameter then it SHALL return a `400 Bad Request` http error code. An OperationOutcome **SHALL** be returned stating that the `lastReceivedOn` parameter is not supported  as shown in the following example:

   {% include lastupdated-notsupported-oo.md %}

##### APIs
{:.no_toc}

In addition to the resources listed above, the following artifacts are used in this transaction:

1. Collect Data operation:[$collect-data]
1. Various DEQM and QI Core Profiles depending on the specific Measure


##### Usage
{:.no_toc}

**Collect Data:**

Using either the `GET` or `POST` Syntax, the operation can be invoked by the Consumer:

`GET|[base]/Measure/[measure-id]/$collect-data&[parameters]`

`POST|[base]/Measure/[measure-id]/$collect-data`

{% include error-note.md transaction = 'Collect Data' %}

{% include examplebutton.html example="collect-data-example" b_title = "Click Here To See Example Collect Data operation (edited for brevity)" %}

For a complete un-edited example see the [COL Collect Data Operation] example.

#### Submit Data and Collect Data for Multiple Patients

##### Submit Data Operation Request for Multiple Patients
{:.no_toc}

The [transaction] bundle processing as defined by FHIR specification is used for transacting the body of Submit Data operation request for *multiple* patients in a single interaction.

- The transaction bundle contains an entry for each patient as illustrated in the examples below:
  - The fullUrl is a UUID (`urn:uuid:...`).
  - The resource is a Parameters resource as defined in the operation.
  - The request method is `POST`
  - The request url is the operation endpoint `Measure/$submit-data` or `Measure/[measure-id]/$submit-data`.
- When resolving references, references are never resolved outside the Parameters resource.  Specifically, resolution stops at the elements Parameters.parameter.resource."
- The matching [transaction response] is returned by the operation endpoint server.

~~~
POST|[base]

{
  "resourceType": "Bundle",
  "type": "transaction",
  "entry": [
    {
      "fullUrl": "urn:uuid:79378cb8-8f58-48e8-a5e8-60ac2755b674",
      "resource": {
        "resourceType": "Parameters",
        "parameter": [
          {
            "name": "measurereport",
            "resource": {
              "resourceType": "MeasureReport",
              ...,
            "name": "resource",
            "resource": {
              "resourceType": "Patient",
              ...,
            [other "resource" parameters]
          ]
        },
        "request": {
          "method": "POST",
          "url": "Measure/[measure-id]/$submit-data"
            }
            ....
~~~

##### Collect Data Operation Response for Multiple Patients
{:.no_toc}

Because operations are typically executed synchronously, a collect data request to a server returns a Parameter resource for a *single* patient as defined by the `$collect-data` operation.  Execution of this operation and returning multiple patients in a single *asynchronous* transaction is outside the scope of this guide.

#### DEQM Data Exchange Operations Naming

The two DEQM data exchange operations, [$submit-data] and [$collect-data], have the same names as two of the base FHIR operations. They serve very similar functions, but the DEQM versions have more flexibility when referencing measures, subjects, and organizations, and they support multiple subjects and measures. To avoid confusion, implementers of the DEQM operations SHOULD NOT support the base FHIR $submit-data and $collect-data operations.

### Provenance

Note that the use of the [X-Provenance header data]({{site.data.fhir.path}}provenance.html#header) with data that establishes provenance being submitted/collected **SHOULD** be supported.  This provides the capability for associating the provider with the data submitted through the $submit-data and $collect-data transactions described above. If the X-Provenance header is used it should be consistent with the `reporter` element in the DEQM Data Exchange MeasureReport Profile.

<br />


{% include link-list.md %}
