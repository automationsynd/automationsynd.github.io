---
layout: post
title:  "Invoke Power BI"
sub_title: "Automating Power BI management with PowerShell"
date:   2019-1-2 8:00:00 -0500
categories: powerbi
author:
    name: Jeremy Brun
    twitter: jeremytbrun
---

### Why Bother?

In my organization we've been leveraging Power BI to picture our data in ways that allows us to make informed business decisions and drive out projects and initiatives effectively.

In a large organization some of your datasources that tell the best story will be within protected network segments that have limited access to the internet. This can present a challenge when you are working with a BI service that is cloud-native.

Of course, Microsoft understands this, and provides the [Power BI Gateway](https://powerbi.microsoft.com/en-us/gateway/) to bridge that gap.

> With the on-premises gateways, you can keep your data fresh by connecting to your on-premises datasources without the need to move the data. Query large datasets and benefit from your existing investments. The gateways provide the flexibility you need to meet individual needs, and the needs of your organization.

The key advantage to using the gateway service in a protected network environment is that it requires *outbound internet access* ***only***.

Specifically, if you would like to poke specific holes in your enterprise firewall for this service only you can get a weekly feed of Azure datacenter IP ranges.

|Product|Region|IP Ranges|
|---|---|---|
|Power BI|Commercial|[Download](https://www.microsoft.com/en-us/download/details.aspx?id=41653)|
|Power BI Gov|Government|[Download](https://www.microsoft.com/en-us/download/details.aspx?id=57063)|

You'll need to open up the following outbound ports to Azure IP ranges so that the gateway can communicate with Azure Service Bus.

- TCP 443 (default)
- 5671
- 5672
- 9350 thru 9354

### Then What?

Once you have Power BI Gateway up and running it is up to you to manage it - whether for yourself or for your customers. In my own experience, I've encountered some challenges in managing such a service once you get beyond even just a handful of datasource connections streaming through the gateway. You mileage may vary, but for me, the UI for managing Power BI Gateway datasources through the online portal quickly became cumbersome and clunky.

Luckly, like any good Microsoft product these days, a [PowerShell module](https://docs.microsoft.com/en-us/powershell/power-bi/overview?view=powerbi-ps) is available for managing Power BI. While it seems to be quite capable in the realm of managing individual dashboards, reports, and datasets at the consumer level, it doesn't currently have very many native cmdlets for managing gateways and corresponding datasource connections. That's not to say it isn't capable of it though, since behind the scenes it's just interacting with the [Power BI REST API](https://docs.microsoft.com/en-us/rest/api/power-bi/).

> *By the way, if you want to help develop the Power BI PowerShell module you can! There is a [GitHub repo](https://github.com/Microsoft/powerbi-powershell) for it.*

Until there are native cmdlets for managing gateway assets, there is an alternative. In the Power BI module there is a generic cmdlet called `Invoke-PowerBIRestMethod`. This is a helper cmdlet that allows you to make any call against the REST API that you want. It handles the passing of OAUTH tokens after you've authenticated to the Power BI API using `Connect-PowerBIServiceAccount`.

### PowerShell Examples

#### Get Connected

```powershell
Connect-PowerBIServiceAccount
# If you need to connect to Power BI for Government you can use this syntax
# Connect-PowerBIServiceAccount -Environment USGov
```

#### Get Gateways

```powershell
$Response = Invoke-PowerBIRestMethod -Url "gateways" -Method Get | ConvertFrom-Json
# Get the first gateway
$Gateway = $Response.value[0]
```

#### Get Datasources for a Gateway

```powershell
$Response = Invoke-PowerBIRestMethod -Url "gateways/$($Gateway.Id)/datasources" -Body $Body -Method Post | ConvertFrom-Json
# Output datasource data
$Response.value
```

#### Create a Datasource

This one is a bit trickier. In order to create a datasource that uses some form of authentication you have to create an encoded credential object. I stumbled across [a post](https://community.powerbi.com/t5/Developer/Automating-Power-BI-Gateway-administration-by-using-Powershell/m-p/212635/highlight/true#M6713) from [Eric Zhang](https://community.powerbi.com/t5/user/viewprofilepage/user-id/6971) on the Power BI community forums wth sample source code for creating such an encoded credential object in C# (half way there!).

So, typically once you have *at least* some C# code you can either reproduce the same behaviour in native PowerShell OR just load the C# class as a type into PowerShell and use it as provided. The latter is the route I chose to take.

Here is the function I created to return an encoded credential.

```powershell
Function Encode-Credential {
    param(
        [String] $Username,
        [String] $Password,
        [String] $GatewayPublicKeyExponent,
        [String] $GatewayPublicKeyModulus
    )

    $Source = @"
    using System;
    using System.Security.Cryptography;
    using System.Text;

    public static class AsymmetricKeyEncryptionHelper
    {

        private const int SegmentLength = 85;
        private const int EncryptedLength = 128;


        /// <summary>
        /// 
        /// </summary>
        /// <param name="userName"></param> the datasouce user name
        /// <param name="password"></param> the datasource password
        /// <param name="gatewaypublicKeyExponent"></param> gateway publicKey Exponent field, you can get it from the get gateways api response json
        /// <param name="gatewaypublicKeyModulus"></param> gateway publicKey Modulus field, you can get it from the get gateways api response json
        /// <returns></returns>
        public static string EncodedCredentials(string userName, string password, string gatewaypublicKeyExponent, string gatewaypublicKeyModulus)
        {
            // using json serializer to handle escape characters in username and password
            {% raw %}var plainText = string.Format("{{\"credentialData\":[{{\"value\":{0},\"name\":\"username\"}},{{\"value\":{1},\"name\":\"password\"}}]}}", userName, password);{% endraw %}

            using (RSACryptoServiceProvider rsa = new RSACryptoServiceProvider(EncryptedLength * 8))
            {
                var parameters = rsa.ExportParameters(false);
                parameters.Exponent = Convert.FromBase64String(gatewaypublicKeyExponent);
                parameters.Modulus = Convert.FromBase64String(gatewaypublicKeyModulus);
                rsa.ImportParameters(parameters);
                return Encrypt(plainText, rsa);
            }
        }

        private static string Encrypt(string plainText, RSACryptoServiceProvider rsa)
        {
            byte[] plainTextArray = Encoding.UTF8.GetBytes(plainText);

            // Split the message into different segments, each segment's length is 85. So the result may be 85,85,85,20.
            bool hasIncompleteSegment = plainTextArray.Length % SegmentLength != 0;

            int segmentNumber = (!hasIncompleteSegment) ? (plainTextArray.Length / SegmentLength) : ((plainTextArray.Length / SegmentLength) + 1);

            byte[] encryptedData = new byte[segmentNumber * EncryptedLength];
            int encryptedDataPosition = 0;

            for (var i = 0; i < segmentNumber; i++)
            {
                int lengthToCopy;

                if (i == segmentNumber - 1 && hasIncompleteSegment)
                    lengthToCopy = plainTextArray.Length % SegmentLength;
                else
                    lengthToCopy = SegmentLength;

                var segment = new byte[lengthToCopy];

                Array.Copy(plainTextArray, i * SegmentLength, segment, 0, lengthToCopy);

                var segmentEncryptedResult = rsa.Encrypt(segment, true);

                Array.Copy(segmentEncryptedResult, 0, encryptedData, encryptedDataPosition, segmentEncryptedResult.Length);

                encryptedDataPosition += segmentEncryptedResult.Length;
            }

            return Convert.ToBase64String(encryptedData);
        }
    }
"@

    Add-Type -TypeDefinition $Source -Language CSharp

    $InputUsername = "$(ConvertTo-Json -InputObject $Username)"
    $InputPassword = "$(ConvertTo-Json -InputObject $Password)"

    return [AsymmetricKeyEncryptionHelper]::EncodedCredentials($InputUsername, $InputPassword, $GatewayPublicKeyExponent, $GatewayPublicKeyModulus)
}
```

And here is an example of creating a new datasource leveraging the `Encode-Credential` function.

```powershell
$Credential = Encode-Credential -Username $Username -Password $Password -GatewayPublicKeyExponent$Gateway.publicKey.exponent -GatewayPublicKeyModulus $Gateway.publicKey.modulus

$Body = @{
    dataSourceType    = "AnalysisServices"
    connectionDetails = "{""server"":""<server>"",""database"":""<database>""}"
    datasourceName    = "New Connection"
    credentialDetails = @{
        credentialType      = "Windows"
        credentials         = $Credential
        encryptedConnection = "Encrypted"
        privacyLevel        = "Private"
        encryptionAlgorithm = "RSA-OAEP"
    }
} | ConvertTo-Json

$Response = Invoke-PowerBIRestMethod -Url "gateways/$($Gateway.Id)/datasources" -Body $Body -Method Post  ConvertFrom-Json
# Get the first datasource
$Datasource = $Response.value[0]
```

#### Add User to a Datasource

```powershell
$Body = @{
    emailAddress          = "<email address>"
    datasourceAccessRight = "Read"
} | ConvertTo-Json

$Response = Invoke-PowerBIRestMethod -Url "gateways/$($Datasource.gatewayId)/datasources/$($Datasource.Id)/users" -Body $Body -Method Post
```

### Wrapping Up

As you can see the management of a Power BI Gateway can easily be automated by digging into the REST API available.

As I encounter new challenges that can be automated and easier managed through PowerShell I will try to add examples of those to this post.