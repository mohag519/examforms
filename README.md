# RIS-3812

For more background information see:

>[Merge request in gitlab](https://gitlab.sectra.net/radiology/RIS/merge_requests/1003)

> [Jira ticket](https://jira.sectra.net/browse/RIS-3812)

---

## Background

Server CAP already provides methods for getting exam form data, now a method to adding data is required. The primary goal is to provide a method to enter screening questionnaire data into Sectra RIS.

Extend the Server CAP interface with methods to create, update and/or delete exam form data for a specific exam form instance and in addition extend the output from the Get exam form data method to optionally include more information regarding the exam form definition, i.e. title, possible values etc

---

## Overview

A brief recap of the implementation in Sectra RIS.  

## Defintions

| Database Table              | Description                                                                                                                     
| ----------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| `TC_FORM_CONDITION`                  | Holds the condition definition as it is defined in the Form Admin.                                                              |
| `TC_FORM_FIELD`                      | Holds an field definition as it is defined in the Form Admin. This is the fields you can use when creating a new form template. |
| `TC_FORM_FIELDVALUE`                   | Holds valid values for a `TC_FORM_FIELD`, i.e. the options for a COMBOBOX.                                                      |
| `TC_FORM_DATA`     | Holds the value saved, connects `TC_EXAMINATION` and `TC_FROM_TEMPLATE`                                                         |
| `TC_FORM_TEMPLATE`          | A form template, this is the actual examination form without data, data is stored *outside* in `TC_FORM_DATA`                   |
| `TC_FORM_TEMPLATE_FIELD`        | A form field template, this is an **instance** of `TC_FORM_FIELD`, holds caption and layout information.                        |
| `TC_FORM_TEMPLATE_TAB`                     | A form template tab is simply a name that may group `TC_FORM_TEMPLATE_FIELD`, each `TC_FORM_TEMPLATE` has one or more tabs      |
| `TC_FORM_TEMPLATE_CONDITION`                     | Connection table that connects `TC_FROM_TEMPLATE` and `TC_FORM_CONDITION`                                                       |
| `TC_FORM_TEMPLATE_EXAMTYPE`                     | Connection table that connects  `TC_FROM_TEMPLATE` with `TC_EXAMTYPE` and `TC_CLINIC`                                           |
| `TC_FORM_TEMPLATE_STATUS`                     | Connection table that connects  `TC_FROM_TEMPLATE` and `TC_CODE_GROUP` (Examination Statuses)                                   |

## Form Admin
Examination Form Admin consists of 5 tabs; `fields`, `conditions`, `forms`, `tabs` and `layout`.

| Tab              | Description 
| -----------------| --------- |
| `fields`         | Administers `TC_FORM_FIELD` and `TC_FORM_FIELDVALUE` |
| `conditions`     | Administers `TC_FORM_CONDITION` |
| `forms`          | Administers `TC_FORM_TEMPLATE` and connections to`TC_FORM_TEMPLATE_CONDITION`, `TC_FORM_TEMPLATE_EXAMTYPE` and `TC_FORM_TEMPLATE_STATUS` |
| `tabs`           | Administers `TC_FORM_TEMPLATE_TAB` and `TC_FORM_TEMPLATE_FIELD` |
| `layout`         | Administers **instances** of `TC_FORM_TEMPLATE_FIELD` and the layout |

The basic principle is that **any connection** prevents deletion, instead the conflicting data may be inactivated. For instance `fields` cannot longer be removed when a `tab` has been created. In the same way when form data has been saved to an examination `TC_FORM_TEMPLATE_FIELD` can no longer be changed or removed.

Once a `tab` is created a set of `TC_FORM_TEMPLATE_FIELD` is created, to change the layout you do that in the `layout` tab. You are free to change the layout at any time, however you want, but please beware as soon there is data saved for a field. Any change (like moving it around, changing the caption etc) will inactivate the old field and create a new active field. Any previously saved data will now be connected to the old inactive field, an inactive field is locked for editing.

The remaining part of this document will describe the actual change in ServerCAP to support this product change.

## Examination form templates
A new controller `ExaminationFormTemplateController` for actions regarding examination form templates. 

The following is an overview:

- Added `Search` with options to search for 
  - Get All, 
  - Get By clinic
  - Get By clinic and examination type
- Added `SearchByExamination` which enforce that the search uses an existing examination.  Using this search gives:
  - All defined templates for the examination (right now)
  - All templates connected to previously saved exam forms for this examination i.e. historical template.
- Added new `DataContracts` as:  
  |- `ExaminationFormTemplates` (act *collection*)  
  &nbsp;&nbsp;&nbsp;&nbsp;|- `ExaminationFormTemplate`  
    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;|- `ExaminationFormConditionTemplate`  
    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;|- `ExaminationFormFieldTemplate`  
       &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;|-  `ExaminationFormTabTemplate`

### Change details 
More details about changes follow
### Added endpoint `SearchByExamination` 
URL `[GET: /CustomerAdaptations/ServerCAP/v2/ExaminationFormTemplates/ByExamination?query]` 

Input options:

|Field Name|Data Type|Default value|Origin|Maps|
| ---------|---------|-------------|------|----|
|examinationNumber|string|none required-field|query URI|TC_EXAMINATION. US_ACCESSION_NR|
|`includeDisabled`|bool|`false`|query URI|
|`includeDisabledFields`|bool|`false`|query URI|

> **Option**  - `IncludeDisabled`    
Controls the available output, once specified also include inactive templates.  
>  
>**Defaults** to `false`

> **Option**  - `IncludeDisabledFields`    
Controls the available output, once specified also include template fields which are inactive.  
>  
>**Defaults** to `false`

Requirements:
- Performing clinic requires hidden site setting 10888 set to enabled.
- Only templates with external IDs are considered.

> **NOTE** A clinic and examination type can have multiple examination form templates defined at the same time both active and inactive.

> **NOTE** Any historical exam forms that has saved data for this examination will be returned as well, regardless if it is connected to the clinic/exam-type any longer and/or if the template is active or inactive.

> **NOTE** Audit logging  
> Write to the audit log `ExaminationFormTemplatesExported` if setting 10979 is specified i.e. `IsExtendedServerCapAuditLoggingEnabled`

Output:  
`ExaminationFormTemplates`

&nbsp;
### Added endpoint `Search`
URL `[GET: /CustomerAdaptations/ServerCAP/v2/ExaminationFormTemplates?query]`

Input options:

|Field Name|Data Type|Default value|Origin|Maps|
| ---------|---------|-------------|------|----|
|clinicGuid|string|null|query URI|TC_CLINIC.KLIN_GUID|
|examinationTypeGuid|string|null|query URI|TC_EXAMTYPE.UST_GUID|
|`includeDisabled`|bool|`false`|query URI|
|`includeDisabledFields`|bool|`false`|query URI|

> **Option**  - `IncludeDisabled`    
Controls the available output, once specified also include inactive templates.  
>  
>**Defaults** to `false`

> **Option**  - `IncludeDisabledFields`    
Controls the available output, once specified also include template fields which are inactive.  
>  
>**Defaults** to `false`

Requirements:
- Specified clinic requires hidden site setting 10888 set to enabled.
- Only templates with external IDs are considered.

> **NOTE** No clinic specified  
> When no clinic is specified the search automatically filter out any clinic that doesn't have the hidden site setting 10888 set to enable rather than raising an exception.

> **NOTE** No clinic but examination type specified  
> This is a illegal combination and will lead to a error message

> **NOTE** Audit logging  
> Write to the audit log `ExaminationFormTemplatesExported` if setting 10979 is specified i.e. `IsExtendedServerCapAuditLoggingEnabled`

Output:  
`ExaminationFormTemplates`

## Changes to data contracts

### Added data contract `ExaminationFormTemplates`
Represents a set of templates

Fields:
|Field Name|Data Type|Maps|
| ---------|---------|----|
|Templates|`IList<ExaminationFormTemplate>`|n/a|

&nbsp;
### Added data contract `ExaminationFormTemplate`
Represent one examination form template

Fields:
|Field Name|Data Type|Maps|
| ---------|---------|----|
|ExternalId|GUID|TC_FORM_TEMPLATE.FMT_EXTERNALID|
|Name|string|TC_FORM_TEMPLATE.FMT_NAME|
|IsActive|bool|TC_FORM_TEMPLATE.FMT_ACTIVE|
|HasData|bool?|n/a|
|IsMandatory|bool|TC_FORM_TEMPLATE.FMT_DATA_REQUIRED|
|EditableStatuses|`IList<ExaminationStatus>`|TC_FORM_TEMPLATE_STATUS.FMTS_COGRID|
|Conditions|`IList<ExaminationFormConditionTemplate>`|TC_FORM_CONDITION|
|Fields|`IList<ExaminationFormFieldTemplate>`|TC_FORM_TEMPLATE_FIELD|
|ValidForClinics|`IList<string>`|n/a|
|ValidForExaminationTypes|`IList<string>`|n/a|

> **Note**  - `HasData`    
This is a convenience field that may be unspecified depending on how the search for this template was made. This field will only be populated with a value (true or false) if the search was initiated on examination number in all other cases it is unspecified.

&nbsp;
### Added data contract `ExaminationFormConditionTemplate`
Represents one condition

Fields:
|Field Name|Data Type|Maps|
| ---------|---------|----|
|Description|string|TC_FORM_CONDITION.COND_DESCRIPTION|
|Gender|Gender?|TC_FORM_CONDITION.COND_SEX|
|AgeMin|int?|TC_FORM_CONDITION.COND_AGE_MIN|
|AgeMax|int?|TC_FORM_CONDITION.COND_AGE_MAX|

&nbsp;
### Added data contract `ExaminationFormFieldTemplate`

Fields:
|Field Name|Data Type|Maps|
| ---------|---------|----|
|Id|string|TC_FORM_TEMPLATE_FIELD.FMTF_ID|
|Name|string|TC_FORM_FIELD.FMF_NAME|
|Caption|string|TC_FORM_TEMPLATE_FIELD.FMTF_CAPTION|
|DefaultValue|string?|TC_FORM_FIELD.FMF_DEFAULTVALUE|
|IsActive|bool|TC_FORM_TEMPLATE_FIELD.FMTF_ACTIVE|
|IsMandatory|bool|TC_FORM_TEMPLATE_FIELD.FMTF_REQUIRED|
|Type|`FieldDatatype`|TC_FORM_FIELD.FMF_TYPE|
|MaxLength|int?|TC_FORM_TEMPLATE_FIELD.FMTF_CHARLENGTH|
|MultiChoiceOptions|`IList<string>`|TC_FORM_FIELDVALUE.FMFV_VALUE|
|Tab|`ExaminationFormTabTemplate`|TC_FORM_TEMPLATE_TAB|

&nbsp;
### Added data contract `ExaminationFormTabTemplate`

Fields:
|Field Name|Data Type|Maps|
| ---------|---------|----|
|Name|string|TC_FORM_TEMPLATE_TAB.FMTT_NAME|
|Order|long|TC_FORM_TEMPLATE_TAB.FMTT_ORDER|
|IsActive|bool|TC_FORM_TEMPLATE_TAB.FMTT_ACTIVE|

## Examination form changes
Extended the implementation of endpoint `ExaminationFormController` and its `DataContracts`, the following is an overview:

- Extended `Search` added grouping and options
- Added `Create` for saving a new examination form
- Added `Update` for updating an existing examination form
- Added `Delete` for deleting an existing examination form
- Added new `DataContracts` as:  
  &nbsp;&nbsp;&nbsp;&nbsp;|- `ExaminationForms` (act *collection*)   
  &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;|- `ExaminationForm`  
    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;|- `ExaminationFormField`

### Change details
More details about change follow.

### Extended endpoint `Search`  
URL: `[GET: /CustomerAdaptations/ServerCAP/v2/ExaminationForms?query]` 

Input options:

|Field Name|Data Type|Default value|Origin|
| ---------|---------|-------------|------|
|`groupByExaminationForms`|bool|`false`|query URI|
|`includeDisabled`|bool|`false`|query URI|
|`includeDisabledFields`|bool|`false`|query URI|
|`includeTemplateDefinitions`|bool|`false`|query URI|
|`includeNewForms`|bool|`false`|query URI|
|`localizedFormatting`|bool|`true`|query URI|
|`disableLocalizedValidation`|bool|`false`|query URI|

> **Option**  - `groupByExaminationForms`    
Controls the available output, once specified groups the output into `ExaminationForms` model instead of the key-value pair like model `ExaminationFormData`. 
>  
>**Defaults** to `false`  
>It is recommended to use this option, however when the examination form should be saved or update via ServerCAP it must be used.

> **Option**  - `IncludeDisabled`    
Controls the available output, once specified also include form data which has an inactive template.  
>  
>**Defaults** to `false`

> **Option**  - `IncludeDisabledFields`    
Controls the available output, once specified also include form data which has an inactive template field.  
>  
>**Defaults** to `false`

> **Option**  - `IncludeTemplateDefinitions`    
Controls whether the output contains any template definitions or not, this option allows by-passing the template endpoint. Once specified embeds the template definitions and template field definitions inside the exam form, normally these fields are `null`
>  
>**Defaults** to `false`

> **Option**  - `IncludeNewForms`    
Controls whether the output contains all new forms as well as saved forms. By specifying this option an empty exam form will be generated and returned even when no previous data have been saved.
>  
> The property `IsNew` will be populated with `true` if the exam form was generated instead of read from the database.
>  
> **NOTE** This form does not yet exists and must be saved using the `create`-endpoint.

> **Option**  - `LocalizedFormatting`    
Controls whether the value is formatted *as we want the input* or raw *as stored in the database*. If this option is requested the value will be formatted according to input requirements, additionally if no value has been set the default value will be used.
> 
>**Defaults** to `true`  
>It is recommended to use this option

> **Option**  - `DisableLocalizedValidation`    
Controls whether the output of `localizedFormatting` is validated or not. Using this option you are sure that data will be returned but it is not guaranteed that it can be saved back using the `update`-endpoint. This option will apply formatting to all fields possible but not raise an exception if old/invalid data is encountered.


Output:
`ExaminationForms`

&nbsp;
### Added `Create` 
URL `[POST: /CustomerAdaptations/ServerCAP/v2/ExaminationForms]` 

Input options:

|Field Name|Data Type|Default value|Origin|
|---------|---------|-------------|------|
| n/a | `ExaminationForm` | none required field | body |

Requirements:

- The exam form payload `ExaminationForm` cannot be `null`
- Each field in `Fields` may only appear once
- The examination indicated by `ExaminatonNumber` **must not** have a previous saved exam form for this template.
- Examination status must be 'Requested (9100)' or above
- Examination cannot be canceled
- Examination cannot be on hold
- Performing clinic requires hidden site setting 10888 set to enabled.
- The impersonating user must have `EditExaminationPermission`
- Template must be active (including template tab)
- Template must have exactly one tab (configuration possible in RIS Client but not allowed)
- Examination status valid according to the configuration
- Conditions must apply for the patient
- All mandatory fields must have a valid value
- All fields must be valid according to their data type
- Rules for creating a new form:
  - New data must reference existing fields according to the current template definition.
  - New data must only reference active template fields.
  - All fields with data must belong to the same examination

> **NOTE** Audit logging  
> Always writes to the audit log `Examination form was saved from ServerCAP` on successful save.

Output:  
`ExaminationForm`

&nbsp;
### Added `Update` 
URL `[PUT: /CustomerAdaptations/ServerCAP/v2/ExaminationForms/{externalId}]` 

Input options:

|Field Name|Data Type|Default value|Origin|
|---------|---------|-------------|------|
| n/a | `ExaminationForm` | none required field | body |
| externalId | *GUID* | none required field | URI |

Requirements:

- The exam form payload `ExaminationForm` cannot be `null`
- Each fields in `Fields` may only appear once
- The examination indicated by `ExaminatonNumber` must have a previous saved exam form for this template.
- Examination status must be 'Requested (9100)' or above
- Examination cannot be canceled
- Examination cannot be on hold
- Performing clinic requires hidden site setting 10888 set to enabled
- Template ID in body payload must match externalId given in the URI
- The impersonating user must have `EditExaminationPermission`
- Template must be active (including template tab)
- Template must have exactly one tab (configuration possible in RIS Client but not allowed)
- Examination status valid according to the configuration
- Conditions must apply for the patient
- All mandatory fields must have a valid value
- All fields must be valid according to their data type
- Rules for updating an existing form
  - May only update already saved fields
  - Inactive fields are locked for updates

> **NOTE** May only update already saved fields   
> This means no additions are allowed, all fields must be added when the exam form first is created.
>  
> In order to get around this limitation using ServerCAP, the integrator may delete the exam form and then save a new. New forms always uses the current template definition. **Beware** that if the template definition has been inactivated, this workaround is not possible.

> **NOTE** Inactive fields are locked for updates   
> The typical reason for this is that a field was active at the time of the exam form creation but has later been inactivated, inactivated fields are locked for editing but an attempt was made to change the field value.

> **NOTE** Audit logging  
> Always writes to the audit log `Examination form was updated from ServerCAP` on successful save.

Output:  
`ExaminationForm`

&nbsp;
### Added `DELETE`
URL `[DELETE /CustomerAdaptations/ServerCAP/v2/ExaminationForms/{externalId}?query]` 

Input options:

|Field Name|Data Type|Default value|Origin|
| ---------|---------|-------------|------|
| externalId | *GUID* | none required field | URI |
| ExaminationNumber | *string* | none required field | query URI |

Requirements:

- The examination indicated by `ExaminatonNumber` must have a previous saved exam form for this template.
- Examination status must be 'Requested (9100)' or above
- Examination cannot be canceled
- Examination cannot be on hold
- Performing clinic requires hidden site setting 10888 set to enabled
- The impersonating user must have `EditExaminationPermission` and `DeleteExaminationFormDataPermission`
- Template must be active (including template tab)
- Template must have exactly one tab (configuration possible in RIS Client but not allowed)
- Examination status valid according to the configuration
- Conditions must apply for the patient

> **NOTE** Audit logging  
> Always writes to the audit log `Examination form was removed from ServerCAP` on successful delete.

Output:  
*empty*

## Changes to data contracts

### Added data contract `ExaminationForms`
Added to allow for a collection of forms to be retrieved when searching for form data, this simplifies the client implementation because the format is the same as what create/update accepts.

Fields:
|Field Name|Data Type|Maps|
| ---------|---------|----|
|Forms|`IList<ExaminationForm>`|n/a|

&nbsp;
### Added data contract `ExaminationForm`
Added a way to express a single form instead of a flattened list of key-value pairs.

Fields:
|Field Name|Data Type|Maps|
| ---------|---------|----|
|ExternalId|GUID|TC_FORM_TEMPLATE.FMT_EXTERNALID|
|ExaminationNumber|string|TC_EXAMINATION.US_ACCESSION_NR|
|Name|string|TC_FORM_TEMPLATE.FMT_NAME|
|IsNew|bool|n/a|
|IsActive|bool|TC_FORM_TEMPLATE.FMT_ACTIVE|
|IsMandatory|bool|TC_FORM_TEMPLATE.FMT_DATA_REQUIRED|
|Definitions|`ExaminationFormTemplate?`|TC_FORM_TEMPLATE.FMT_EXTERNALID|
|Fields|`IList<ExaminationFormField>`|n/a|

&nbsp;
### Added data contract `ExaminationFormField`

Fields:
|Field Name|Data Type|Maps|
| ---------|---------|----|
|Id|string|TC_FORM_TEMPLATE_FIELD.FMTF_ID|
|Name|string|TC_FORM_FIELD.FMF_NAME|
|Caption|string|TC_FORM_TEMPLATE_FIELD.FMTF_CAPTION|
|IsActive|bool|TC_FORM_TEMPLATE_FIELD.FMTF_ACTIVE|
|IsMandatory|bool|TC_FORM_TEMPLATE_FIELD.FMTF_REQUIRED|
|InstanceValue|string|TC_FORM_DATA.FMD_VALUE|
|Definitions|`ExaminationFormFieldTemplate?`|TC_FORM_TEMPLATE_FIELD|
|DataType|FieldDatatype|TC_FORM_FIELD.FMF_TYPE|

