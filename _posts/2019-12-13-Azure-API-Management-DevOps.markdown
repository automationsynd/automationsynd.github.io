---
layout: post
title:  "Managing Azure API Management with Infrastructure as Code"
date:   2019-12-13 09:00:00 -0500
categories: devops
---

- [Why Bother?](#why-bother)
    - [Noteworthy community contributions](#noteworthy-community-contributions)
- [Azure API Management DevOps SDK](#azure-api-management-devops-sdk)
- [My Approach](#my-approach)
  - [Get the API spec definitions](#get-the-api-spec-definitions)
  - [Generate and Deploy ARM templates](#generate-and-deploy-arm-templates)
  - [Wrapping it all together](#wrapping-it-all-together)

# Why Bother?

Recently I was tasked with building out a repository and CI/CD process for the Azure API Management (APIM) service and configurations which my organization is using. I thought to myself, "Meh, you've seen one ARM template and you've seen them all!". Well, if you've seen the [APIM template schema](https://docs.microsoft.com/en-us/azure/templates/microsoft.apimanagement/allversions) you're probably snickering right now. I quickly found myself underwater and struggling to keep all the moving parts of an APIM configuration via ARM template. Naturally I did some Google-fu stretches and set out to find someone else who has had the same problem and some viable approaches that may already exist.

Enter the [Azure API Management DevOps SDK](https://github.com/Azure/azure-api-management-devops-resource-kit). The folks over at Microsoft recognized the need for a streamlined template management process and that folks in the community had even started developing their own solutions.

### Noteworthy community contributions

- [Mattias Lögdberg's API Management ARM Template Creator](http://mlogdberg.com/apimanagement/arm-template-creator)
- [Eldert Grootenboer's blog post series on CI/CD with API Management](https://blog.eldert.net/api-management-ci-cd-using-arm-templates-api-management-instance/)

# Azure API Management DevOps SDK

This toolkit is a C# tool with two parts - a **Creator** and **Extractor**.

**Creator** makes Developers and DevOps Engineers not have to worry about the actual ARM template syntax, and only on working with the actual [Swagger/OpenAPI](https://swagger.io/specification/) spec definitions they are implementing. The tool will take a spec definition as input and dynamically create the ARM templates required for deploying into Azure.

**Extractor** is a complementary tool that allows you to target an existing APIM instance and dynamically create the ARM templates based on an existing API configuration. This is extremely valuable for promoting configurations from a dev environment into a demo or staging environment.

# My Approach

So armed with an idea and a whole lot of tenacity I set out to accomplish my task. I had a high-level idea of the order of operations for using ARM templates to deploy APIM into my dev environment, as follows.

## Get the API spec definitions

My organizaton uses [SwaggerHub](https://app.swaggerhub.com/) to design our private API definitions. We decided that this would be the source of truth for what we expose to our consumers within APIM so the first challenge was to figure out how we'd programmatically retrieve those definitions.

Very quickly we landed in the SwaggerHub documentation on [downloading OpenAPI definitions](https://app.swaggerhub.com/help/apis/downloading-swagger-definition). We found that the SwaggerHub API supports downloading private API definitions but ***only the "unresolved" definition***. If you're using `$refs` in your spec definition that link to other external private definitions like us then that presents a challenge. We needed the "resolved" definition which is essentially flattened to include everything in one file. After submitting a support request to the vendor we received a confirmation that the only supported method of retrieving a resolved definition direclty from SwaggerHub is via the GUI...

![Alt Text](https://media.giphy.com/media/6yRVg0HWzgS88/giphy.gif)

OK. Alrighty then. That isn't super great, but thankfully in their response they gave us some breadcrumbs for a potential workaround to this limitation. Enter the open-source [Swagger API Swagger Codegen](https://github.com/swagger-api/swagger-codegen) library. The vendor suggested we look at either the CLI or Maven plugins available for this library which include the functionality to generate an API spec definition file based on a spec URL provided (think SwaggerHub API spec URL). And best of all, the output is a ***resolved*** definition! By the way, you can also use this tool to generate client and server code in many different languages.

We aren't currently using Maven at all so we ruled out that plugin as immediatly worthwhile exploring. I tested the CLI plugin which basically just requires you have the Jave Runtime Environment in order to run it. It worked great an all, but I didn't like the idea of maintaining the local JAR package as well as the JRE on our build server - plus, I mean, it's *Java* right? So... anything else? Yes, indeed there is. Right in the *swagger-codegen* readme it talks about a [hosted version of the generator available as a public API](https://github.com/swagger-api/swagger-codegen#online-generators).

This online generator seemed like the perfect fit. The following is PowerShell script and accompanying configuration file I put together to prove the concept.

***APIMIntegrationConfig.json***
```Json
[
    {
        "ApiVersion": "v3", // API Version from SwaggerHub
        "ApiName": "api-name" // API Name from SwaggerHub
    }
]
```

***Get-SwaggerHubApiDefinition.ps1***
```PowerShell
$SwaggerGeneratorUri = "https://generator3.swagger.io/api/generate"
$SwaggerHubApiBaseUri = "https://api.swaggerhub.com/apis/OWNER"
$SwaggerHubApiKey = "KEY-GUID-HERE"

$SwaggerDefinitionsPathWorking = ".\SwaggerDefinitionsWorking"
$SwaggerDefinitionsPath = ".\SwaggerDefinitions"
$APIMIntegrationConfigFile = ".\APIMIntegrationConfig.json"
$APIMIntegrationConfig = Get-Content -Path $APIMIntegrationConfigFile | ConvertFrom-Json

# Create a working directory to download zipped definitions to
if (-not (Test-Path -Path $SwaggerDefinitionsPathWorking)) {
    New-Item -Path $SwaggerDefinitionsPathWorking -ItemType Directory | Out-Null
}

# Verify the working directory is empty
Get-ChildItem -Path $SwaggerDefinitionsPathWorking -Force | Remove-Item -Force

# Initialize the published definition folder
if(-not (Test-Path -Path $SwaggerDefinitionsPath)) {
    New-Item -Path $SwaggerDefinitionsPath -ItemType Directory | Out-Null
}

# Loop through configurations
$APIMIntegrationConfig | ForEach-Object {
    $Api = $_
    $ApiName = $Api.ApiName
    $ApiVersion = $Api.ApiVersion
    $ApiUri = "$SwaggerHubApiBaseUri/$ApiName/$ApiVersion"
    $SwaggerGeneratorOutputZipPath = "$SwaggerDefinitionsPathWorking\SwaggerGenerator.zip"
    $SwaggerGeneratorOutputPath = $SwaggerGeneratorOutputZipPath.TrimEnd(".zip")
    
    $Header = @{
        "Content-Type" = "application/json"
        Accept         = "application/octet-stream"
    }

    $Body = @{
        lang           = "swagger"
        specURL        = $ApiUri
        type           = "SERVER"
        codegenVersion = "V2"
        options        = @{
            auth = "Authorization:$SwaggerHubApiKey"
        }
    } | ConvertTo-Json -Depth 100

    # Download the zipped definition
    Invoke-RestMethod -Method Post -Headers $Header -Uri $SwaggerGeneratorUri -Body $Body -OutFile $SwaggerGeneratorOutputZipPath

    # Expand in working directory
    Expand-Archive -Path $SwaggerGeneratorOutputZipPath -DestinationPath $SwaggerGeneratorOutputPath -Force

    # Copy the definition file to the published folder
    Copy-Item -Path "$SwaggerGeneratorOutputPath\swagger.json" -Destination "$SwaggerDefinitionsPath\$($ApiName)_swagger.json" -Force

    #Clear the working directory
    Get-ChildItem -Path $SwaggerDefinitionsPathWorking -Force | Remove-Item -Recurse -Force
}
```

## Generate and Deploy ARM templates

Now that we have an API definition file downloaded we can invoke the APIM DevOps SDK Creator tool to build ARM templates for us! The Creator tool also uses a YAML file to configure APIM specific settings for each API such as products, tags, and many other optional settings. The implementation for that YAML file will vary greatly on the implemenation and organization needs.

I have all of the files above as well as the APIM DevOps SDK source code all in my APIM repo. In my build pipeline I use the .NET Core CLI restore, build, test, and run tasks to run the Creator tool. This tool creates ARM templates which can be published as artifacts and then consumed and deployed by a release pipeline.

## Wrapping it all together

So far we are very happy with the end-product of our proof-of-concept. What this means for developers and product owners is the ability to merely change a version in the `APIMIntegrationConfig.json` file when new versions of an existing API are designed in SwaggerHub, implemented in code, then released. This would kick off our build and release pipelines which would publish the updated API definition to APIM.

For new versions of an API there are a few more steps required which could probably be fine-tuned. Assuming the definition has been designed in SwaggerHub and it has been implemented in code then a new entry would have to be entered in the `APIMIntegrationConfig.json` file as well as new configuration data added to the APIM DevOps SDK Creator YAML file.

But that all being said, the clear advantage is that now you can maintain your APIM configuration within a repository via Infrastructure as Code. It doesn't take much additional effort to also include your APIM instance template as well.