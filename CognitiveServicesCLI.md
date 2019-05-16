# Azure CLI Installation
1. Download & Install: https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest
2. Manage subscription: https://docs.microsoft.com/en-us/cli/azure/manage-azure-subscriptions-azure-cli?view=azure-cli-latest
3. Manage resource group: https://docs.microsoft.com/en-us/cli/azure/group?view=azure-cli-latest

# Cognitive Services Deployment through CLI

|Step|Command|
|-|-|
|Create resource group|`az group create -l EastAsia -n apiResourceGroup`|
|Create text analytics|`az cognitiveservices account create -n textanalyticsapi -g apiResourceGroup --kind TextAnalytics --sku S3 -l EastAsia --y`|
|List text analytics keys|`az cognitiveservices account keys list --name textanalyticsapi -g apiResourceGroup `|
|Pull text analytics image|`docker pull mcr.microsoft.com/azure-cognitive-services/keyphrase:latest`|
|Run analytics image|`docker run --rm -it -p 5000:5000 --memory 4g --cpus 1 mcr.microsoft.com/azure-cognitive-services/keyphrase Eula=accept Billing=https://eastasia.api.cognitive.microsoft.com/text/analytics/v2.0 ApiKey={BILLING_KEY}`

Test it:

`curl --request POST \
  --url http://localhost:5000/text/analytics/v2.0/sentiment \
  --header 'cache-control: no-cache' \
  --header 'content-type: application/json' \
  --data '{"documents": [{"language": "en","id": "1","text": "Hello world. This is some input text that I love."}]}'`

## Reference

- Cognitive Services Azure CLI reference: CLI reference: https://docs.microsoft.com/en-us/cli/azure/cognitiveservices?view=azure-cli-latest#az_cognitiveservices_list