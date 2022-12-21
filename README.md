# Adventures with CQL

These are notes from some first adventures with HL7's Clinical Query Language
([CQL](https://cql.hl7.org/)).  We used NICE guideline NG28 (*Type 2 diabetes
in adults: management*), [section 1.7](https://www.nice.org.uk/guidance/ng28/chapter/Recommendations#drug-treatment)
as a context, though the main activity was very much "getting to know CQL".

We used https://cql-runner.dataphoria.org/ to query https://hapi.fhir.org/ following suggestions from https://github.com/Prayance/Tutorial-CQL.

we created three example patients and then ran this CQL query, loosely connected to the guideline:

```CQL
library myLibrary version '1'
//All libraries and CQL expressions shall use FHIR based data models and you need to specify the version.
using FHIR version '4.0.1'

/*
  FHIRHelpers is a common asset library details at: http://fhir.org/guides/cqf/common/Library-FHIRHelpers.html
  If this does not work in dataphoria engine, remove the version and use
  include FHIRHelpers called FHIRHelpers
*/
include FHIRHelpers version '4.0.1' called FHIRHelpers

codesystem "SNOMED": 'http://snomed.info/sct'
codesystem "LOINC": 'https://loinc.org/'
codesystem "ConditionCategoryCodes": 'http://terminology.hl7.org/CodeSystem/condition-category'
codesystem "ObservationCategoryCodes": 'http://terminology.hl7.org/CodeSystem/observation-category'
codesystem "ConditionClinicalStatusCodes": 'http://terminology.hl7.org/CodeSystem/condition-clinical'

valueset "Active Condition": 'http://fhir.org/guides/cqf/common/ValueSet/active-condition'
valueset "Inactive Condition": 'http://fhir.org/guides/cqf/common/ValueSet/inactive-condition'

code "active": 'active' from "ConditionClinicalStatusCodes" display 'active'
code "Type 2 Diabetes": '44054006' from "SNOMED" display 'Diabetes mellitus type II'
code "Metabolic acidosis": '59455009' from "SNOMED" display 'Metabolic acidosis'
code "Diabetic metabolic acidosis": '735539005' from "SNOMED" display 'Diabetic metabolic acidosis'
code "Acidosis due to type 2 diabetes": '721284006' from "SNOMED" display 'Acidosis due to type 2 diabetes mellitus'
code "Diabetic lactic acidosis": '735538002' from "SNOMED" display 'Diabetic lactic acidosis'
code "Diabetic ketoacidosis": '420422005' from "SNOMED" display 'Diabetic ketoacidosis'
code "Metformin": '109081006' from "SNOMED" display 'Metformin'
code "Birthdate": '21112-8' from "LOINC" display 'Birth date'


parameter MeasurementPeriod Interval<DateTime>
    default Interval[@2019-01-01T00:00:00.0, @2022-10-10T00:00:00.0)

context Patient

define "prescribedMetformin":
  exists(PrescribedDrug([MedicationDispense: "Metformin"]))

define "Diagnosis":
    [Condition]

define "Prescriptions":
    [MedicationDispense]

define "InDemographic":
	AgeInYearsAt(start of MeasurementPeriod) >= 18

define "InScope":
    if "InDemographic" then 'This patient could need Metformin'
    else 'This patient is not included in the cohort'

define "hasDiabetes":
  exists(ActiveCondition([Condition: "Type 2 Diabetes"]))

define "hasMetabolicAcidosis":
  exists(ActiveCondition([Condition: "Acidosis due to type 2 diabetes"])) or
  	exists(ActiveCondition([Condition: "Metabolic acidosis"])) or
      exists(ActiveCondition([Condition: "Diabetic metabolic acidosis"])) or
        exists(ActiveCondition([Condition: "Diabetic lactic acidosis"])) or
          exists(ActiveCondition([Condition: "Diabetic ketoacidosis"]))

define function ActiveCondition(CondList List<Condition>):
  CondList C
  	where C.clinicalStatus ~ "active"

define function PrescribedDrug(MedList List<MedicationDispense>):
  MedList M

define "prescribeMetformin":
  if "hasDiabetes" and not "hasMetabolicAcidosis" then 'Prescribe Metformin'
  else 'Do not prescribe Metformin'
```

Note that to run the query you will need to set the "Data Source URL" in the cql-runner config dialogue to ``http://hapi.fhir.org/baseR4`` and you'll need to set the Patient ID to one of the created patient IDs.  The results that we saw for each of the three example patients are in the results.*.txt files.

## setup

If running the CQL query does not provide the results you expect then the data
may need to be setup which you can do manually or with the setup.js script.

### manually

1. point your browser to [hapi.fhir.org/Patient](http://hapi.fhir.org/resource?serverId=home_r4&pretty=true&_summary=&resource=Patient)
2. click the "CRUD Operations" tab
3. enter the p1.json content into Create > contents text box and click the associated create button.
4. note down the generated id
5. update p1d1.json and p1d2.json subject.reference attribute with the newly created id
6. point your browser to [hapi.fhir.org/Condition](http://hapi.fhir.org/resource?serverId=home_r4&pretty=true&_summary=&resource=Condition)
7. click the CRUD Operations tab
8. paste the contents of p1d1.json into the create box and submit
9. note the id of the generated condition (for cleaning up later)
10. repeat for p1d2.json

Repeat this process for patient two and patient three.  Note that patient three has four prescriptions for which you'll need the [hapi.fhir.org/MedicationDispense](http://hapi.fhir.org/resource?serverId=home_r4&pretty=true&_summary=&resource=MedicationDispense) content type.

### Using node

TODO: With nodejs installed you can run ``node setup.js`` at
the commandline and hopefully see the following:

```
```

You should also see a new text file, patients.txt in the folder with the ids
that you have created to make it easier to clean up after yourself.

## cleaning up

hapi.fhir.org is a public server so you may wish to clear up after yourself.  If you dont do this, the database will likely be automatically reset at somepoint.

### manually

To manually do this you'll need the ids of the documents you created in the setup.  If you dont have them to hand then you can search for the relevant document.  For ``Patient``s, the dob and name are sufficient and for the ``Condition`` and ``MedicationDispense`` the id of the patient are sufficient.  Then with these ids you can use the delete section of the CRUD operations tab (using the links from the setup section) to delete the documents that you created.

### using node

TODO: You can also use the script clearup.js to clear up after yourself if you used the setup.js script to add the data and it created a text file of ids for you.  
