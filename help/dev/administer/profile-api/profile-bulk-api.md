---
title: Adobe Target Bulk Profile Update API
description: Learn how to use [!DNL Adobe Target] [!UICONTROL Bulk Profile Update API] to send multiple visitors' profile data to [!DNL Target] for use in targeting.
feature: APIs/SDKs
contributors: https://github.com/icaraps
exl-id: 0f38d109-5273-4f73-9488-80eca115d44d
---
# [!DNL Adobe Target Bulk Profile Update API]

The [!DNL Adobe Target] [!UICONTROL Bulk Profile Update API] lets you update user profiles for multiple visitors to a website in bulk using a batch file.

Using the [!UICONTROL Bulk Profile Update API], you can conveniently send detailed visitor profile data in the form of profile parameters for many users to [!DNL Target] from any external source. External sources can include Customer Relationship Management (CRM) or Point of Sale (POS) systems, which are not usually available on a web page.

|Version|URL example|Features|
| --- | --- | --- |
|v1|`http://CLIENTCODE.tt.omtrdc.net/m2/CLIENTCODE/profile/batchUpdate`|Support for bulk profile update only.|
|v2|`http://CLIENTCODE.tt.omtrdc.net/m2/CLIENTCODE/v2/profile/batchUpdate`|<ul><li>Create profile if not found.</li><li>Per row status update.</li></ul>|

>[!NOTE]
>
>Version 2 (v2) of the [!DNL Bulk Profile Update API] is the current version. However, [!DNL Target] continues to support version 1 (v1).
>
>* If your [!DNL Target] implementation uses [!DNL Experience Cloud ID] (ECID) as one of the profile identifiers for anonymous visitors, do not use `pcId` as the key in a Version 2 (v2) batch file. The use of `pcId` with v2 of the [!DNL Bulk Profile Update API] is intended only for stand-alone [!DNL Target] implementations that do not rely on ECID.
>
>* If your implementation uses ECID for profile identification and you want to use `pcId` as the key in the batch file, use Version 1 (v1) of the API.
>
>* If your implementation uses `thirdPartyId` for profile identification, use Version 2 (v2) of the API with `thirdPartyId` as the key.

## Benefits of the [!UICONTROL Bulk Profile Update API]

* No limit on the number of profile attributes.
* Profile attributes sent via the site can be updated via the API and the opposite way.

## Caveats

* The size of the batch file must be less than 50 MB. In addition, the total number of rows should not exceed 500,000 rows per upload.
* Updates generally occur in under one hour, but might take as long as 24 hours to be reflected.
* There is no limit on the number or rows that you can upload over a period of 24 hours in subsequent batches. However, the ingestion process might be throttled during business hours to ensure that other processes run efficiently.
* Consecutive v2 batch update calls without mbox calls in between for the same thirdPartyIds override the properties updated in the first batch update call.
* [!DNL Adobe] does not guarantee that 100% of batch profile data will be onboarded and retained in Target and, thus, be available for use in targeting. In the current design, there is a possibility that a small percentage of data (up to 0.1% of large production batches) might not be onboarded or retained.

## Batch file

To update profile data in bulk, create a batch file. The batch file is a text file with values separated by commas similar to the following sample file.

```
batch=pcId,param1,param2,param3,param4
123,value1
124,value1,,,value4
125,,value2
126,value1,value2,value3,value4
```

>[!NOTE]
>
>The `batch=` parameter is required and must be specified at the beginning of the file.

You reference this file in the POST call to [!DNL Target] servers to process the file. When creating the batch file, consider the following:

* The first row of the file must specify column headers.
* The first header should either be a `pcId` or `thirdPartyId`. The [!UICONTROL Marketing Cloud visitor ID] is not supported. [!UICONTROL pcId] is a [!DNL Target]-generated visitorID. `thirdPartyId` is an ID specified by the client application, which is passed to [!DNL Target] through an mbox call as `mbox3rdPartyId`. It must be referred to here as `thirdPartyId`.
* Parameters and values you specify in the batch file must be URL-encoded using UTF-8 for security reasons. Parameters and values can be forwarded to other edge nodes for processing through HTTP requests.
* The parameters must be in the format `paramName` only. Parameters are displayed in [!DNL Target] as `profile.paramName`.
* If you are using [!UICONTROL Bulk Profile Update API] v2, you need not specify all parameter values for each `pcId`. Profiles are created for any `pcId` or `mbox3rdPartyId` that is not found in [!DNL Target]. If you are using v1, profiles are not created for missing pcIds or mbox3rdPartyIds. For more information, see [Handling empty values in the [!DNL Bulk Profile Update API]](#empty) below.
* The size of the batch file must be less than 50 MB. In addition, the total number of rows should not exceed 500,000. This limit ensures that servers don't get flooded with too many requests.
* There is no restriction on the number of attributes you can upload. However, the total size of the external profile data, which includes Customer Attributes, Profile API, In-Mbox profile parameters, and Profile Script output, must not exceed 64 KB.
* Parameters and values are case-sensitive.

## HTTP POST request

Make an HTTP POST request to [!DNL Target] edge servers to process the file. Here is a sample HTTP POST request for the file batch.txt using the curl command:

```
curl -X POST --data-binary @BATCH.TXT http://CLIENTCODE.tt.omtrdc.net/m2/CLIENTCODE/v2/profile/batchUpdate
```
```
curl --request POST \
  --data-binary @BATCH.TXT
  --url http://<CLIENTCODE>.tt.omtrdc.net/m2/<CLIENTCODE>/v2/profile/batchUpdate \
  --header 'Authorization: Bearer <Token>' \
  --header 'Content-Type: application/x-www-form-urlencoded' \
```
Where:

BATCH.TXT is the filename. CLIENTCODE is the [!DNL Target] client code.

If you don't know your client code, in the [!DNL Target] user interface click **[!UICONTROL Administration]** > **[!UICONTROL Implementation]**. The client code is shown in the [!UICONTROL Account Details] section.

### Inspect the response

The Profiles API returns the submission status of the batch for processing along with a link under "batchStatus" to a different URL that shows the overall status of the particular batch job.

### Example API response

The following code snipped is an example of a Profiles API response:

```
<response>
    <success>true</success>
    <batchStatus>http://mboxedge45.tt.omtrdc.net/m2/demo/profile/batchStatus?batchId=demo-1701473848678-13029383</batchStatus>
    <message>Batch submitted for processing</message>
</response>
```

If there is an error, the response contains `success=false` and a detailed message for the error.

### Default batch status response

A successful default response when the above `batchStatus` URL link is clicked looks like the following:

```
<response><batchId>demo4-1701473848678-13029383</batchId><status>complete</status><batchSize>1</batchSize></response>
```

Expected values for the status fields are:

|Status|Details|
| --- | --- |
|[!UICONTROL complete]|The profile batch update request was completed successfully.|
|[!UICONTROL incomplete]|The profile batch update request is still being processed and not completed.|
|[!UICONTROL stuck]|The profile batch update request is stuck and was not able to complete.|

### Detailed batch status URL response

A more detailed response can be fetched by passing a parameter `showDetails=true` to the `batchStatus` url above.

For example:

```
http://mboxedge45.tt.omtrdc.net/m2/demo/profile/batchStatus?batchId=demo-1701473848678-13029383&showDetails=true
```

#### Detailed response

```
<response>
    <batchId>demo4-1701473848678-13029383</batchId>
    <status>complete</status>
    <batchSize>1</batchSize>
    <consumedCount>1</consumedCount>
    <successfulUpdates>1</successfulUpdates>
    <profilesNotFound>0</profilesNotFound>
    <failedUpdates>0</failedUpdates>
</response>
```

## Handling empty values in the [!DNL Bulk Profile Update API] {#empty}

When using the [!DNL Target] [!DNL Bulk Profile Update API] (v1 or v2), it's important to understand how the system handles empty parameter or attribute values.

### Expected behavior

Sending empty values ("", null, or missing fields) for existing parameters or attributes does not reset or delete those values in the profile store. This is by design.

* **Empty values are ignored**: The API filters out empty values during processing to avoid unnecessary or meaningless updates.

* **No clearing of existing data**: If a parameter already has a value, sending an empty value leaves it unchanged.

* **Empty-only batches are skipped**: If a batch contains only empty or null values, it is ignored entirely and no updates are applied.

### Additional Notes

This behavior applies to both v1 and v2 of the [!DNL Bulk Profile Update API].

Attempting to clear or remove an attribute by sending an empty value has no effect.

Support for explicit attribute removal is planned for a future version (v3) of the API, but is not currently available.
