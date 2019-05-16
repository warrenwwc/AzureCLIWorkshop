# Cognitive Search Deployment through CLI & REST API

## Get key and URL

|Step|Command|Remark|
|-|-|-|
|Create resource group|`az group create -l EastAsia -n searchResourceGroup`|
|Create Azure Search|`az search service create -n <Search Service Name> -g searchResourceGroup -l EastAsia --sku Standard3`|Search Endpoint is https://[Global Unique Name].search.windows.net|
|List search admin key|`az search admin-key show --service-name <Search Service Name> -g searchResourceGroup`|
|Create blob storage for image data|`az storage account create --name <Storage Account Name> --resource-group searchResourceGroup --location eastasia --sku Standard_LRS`|Create Blob for Cognitive Search data source|
|List storage account keys|`az storage account keys list --account-name <Storage Account Name>`|
|Create storage container|`az storage container create --name testdata --account-name <Storage Account Name> --account-key <Storage Account Key>`|
|Upload blob|`az storage blob upload --container-name mystoragecontainer --name blobName --file ~/path/to/local/file`|Alternative - Azure storage explorer: https://azure.microsoft.com/en-us/features/storage-explorer/|

## Create a data source

`curl --request POST \
  --url 'https://<Search Service Name>.search.windows.net/datasources?api-version=2019-05-06' \
  --header 'api-key: <Search Admin Key>' \
  --header 'cache-control: no-cache' \
  --header 'content-type: application/json' \
  --data '{"name" : "demodata","description" : "Demo files to demonstrate cognitive search capabilities.","type" : "azureblob","credentials" :{ "connectionString":"DefaultEndpointsProtocol=https;AccountName=searchdatastorage;AccountKey=008gSqSmRIw8OP4jd/Bm2DNCYPcpN0OW+SkvzB825W9IS0dolPm3mm740J7OaeOWELYEJu9MYklUxTvU2klAzg==;"},"container" : { "name" : "testdata" }}'`

Send the request. It should return a status code of 201 confirming success.

## Create a skillset

In this step, you define a set of enrichment steps that you want to apply to your data. You call each enrichment step a skill, and the set of enrichment steps a skillset. This tutorial uses built-in cognitive skills for the skillset:

- Language Detection to identify the content's language.

- Text Split to break large content into smaller chunks before calling the key phrase extraction skill. Key phrase extraction accepts inputs of 50,000 characters or less. A few of the sample files need splitting up to fit within this limit.

- Entity Recognition for extracting the names of organizations from content in the blob container.

- Key Phrase Extraction to pull out the top key phrases.

This request creates a skillset. Reference the skillset name `demoskillset` for the rest of this tutorial.

`curl --request PUT \
  --url 'https://<Search Service Name>.search.windows.net/skillsets/demoskillset?api-version=2019-05-06' \
  --header 'api-key: <Search Admin Key>' \
  --header 'cache-control: no-cache' \
  --header 'content-type: application/json' \
  --data '{"description":"Extract entities, detect language and extract key-phrases","skills":[{"@odata.type":"#Microsoft.Skills.Text.EntityRecognitionSkill","categories":["Organization"],"defaultLanguageCode":"en","inputs":[{"name":"text","source":"/document/content"}],"outputs":[{"name":"organizations","targetName":"organizations"}]},{"@odata.type":"#Microsoft.Skills.Text.LanguageDetectionSkill","inputs":[{"name":"text","source":"/document/content"}],"outputs":[{"name":"languageCode","targetName":"languageCode"}]},{"@odata.type":"#Microsoft.Skills.Text.SplitSkill","textSplitMode":"pages","maximumPageLength":4000,"inputs":[{"name":"text","source":"/document/content"},{"name":"languageCode","source":"/document/languageCode"}],"outputs":[{"name":"textItems","targetName":"pages"}]},{"@odata.type":"#Microsoft.Skills.Text.KeyPhraseExtractionSkill","context":"/document/pages/*","inputs":[{"name":"text","source":"/document/pages/*"},{"name":"languageCode","source":"/document/languageCode"}],"outputs":[{"name":"keyPhrases","targetName":"keyPhrases"}]}]}'`

Send the request. It should return a status code of 201 confirming success.

## Create an index

This request creates an index. Use the index name `demoindex` for the rest of this tutorial.

`curl --request PUT \
  --url 'https://<Search Service Name>.search.windows.net/indexes/demoindex?api-version=2019-05-06' \
  --header 'api-key: <Search Admin Key>' \
  --header 'cache-control: no-cache' \
  --header 'content-type: application/json' \
  --data '{"fields":[{"name":"id","type":"Edm.String","key":true,"searchable":true,"filterable":false,"facetable":false,"sortable":true},{"name":"content","type":"Edm.String","sortable":false,"searchable":true,"filterable":false,"facetable":false},{"name":"languageCode","type":"Edm.String","searchable":true,"filterable":false,"facetable":false},{"name":"keyPhrases","type":"Collection(Edm.String)","searchable":true,"filterable":false,"facetable":false},{"name":"organizations","type":"Collection(Edm.String)","searchable":true,"sortable":false,"filterable":false,"facetable":false}]}'`

Send the request. It should return a status code of 201 confirming success.

## Create an indexer, map fields, and execute transformations

So far you have created a data source, a skillset, and an index. These three components become part of an indexer that pulls each piece together into a single multi-phased operation. To tie these together in an indexer, you must define field mappings.

- The fieldMappings are processed before the skillset, mapping source fields from the data source to target fields in an index. If field names and types are the same at both ends, no mapping is required.

- The outputFieldMappings are processed after the skillset, referencing sourceFieldNames that don't exist until document cracking or enrichment creates them. The targetFieldName is a field in an index.

`curl --request PUT \
  --url 'https://<Search Service Name>.search.windows.net/indexers/demoindexer?api-version=2019-05-06' \
  --header 'api-key: <Search Admin Key>' \
  --header 'cache-control: no-cache' \
  --header 'content-type: application/json' \
  --data '{"name":"demoindexer","dataSourceName":"demodata","targetIndexName":"demoindex","skillsetName":"demoskillset","fieldMappings":[{"sourceFieldName":"metadata_storage_path","targetFieldName":"id","mappingFunction":{"name":"base64Encode"}},{"sourceFieldName":"content","targetFieldName":"content"}],"outputFieldMappings":[{"sourceFieldName":"/document/organizations","targetFieldName":"organizations"},{"sourceFieldName":"/document/pages/*/keyPhrases/*","targetFieldName":"keyPhrases"},{"sourceFieldName":"/document/languageCode","targetFieldName":"languageCode"}],"parameters":{"maxFailedItems":-1,"maxFailedItemsPerBatch":-1,"configuration":{"dataToExtract":"contentAndMetadata","imageAction":"generateNormalizedImages"}}}'`

## Check indexer status

`curl --request GET \
  --url 'https://<Search Service Name>.search.windows.net/indexers/demoindexer/status?api-version=2019-05-06' \
  --header 'api-key: <Search Admin Key>' \
  --header 'cache-control: no-cache'`

## Test Cognitive Search

`curl --request GET \
  --url 'https://<Search Service Name>.search.windows.net/indexes/demoindex/docs?search=*&api-version=2019-05-06' \
  --header 'api-key: <Search Admin Key>' \
  --header 'cache-control: no-cache'`

## Reference

- Predefined skills for content enrichment (Azure Search): https://docs.microsoft.com/en-us/azure/search/cognitive-search-predefined-skills
- Azure storage explorer: https://azure.microsoft.com/en-us/features/storage-explorer/
- Azure blob sdk: https://docs.microsoft.com/en-us/azure/storage/blobs/storage-quickstart-blobs-nodejs-v10