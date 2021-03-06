---
layout: post
title: Creating an Azure SAS token in F#
description: >
  A walkthrough of creating an Azure SAS token programmatically in F#
redirect_from:
   - /post/creating-an-azure-sas-token-in-f
---

I wanted to be able to publish events from an IoT application. I could not figure out how to create the SAS token in the language of the IoT application so I decided to just create one that would last for 90 days.

The first thing you will need is an Event Hub in Azure. Get the URL you are going to use for sending events. It should look like this:

>https://youreventhub-ns.servicebus.windows.net/youreventhub/publishers/test01/messages

You will need to create a shared access policy that will allow you to send events to the Event Hub.

![shared access policies]({{ baseurl }}/post_images/2015/08/shared_access_policies.png)

Then get the key from the policy.

![shared access key generator]({{ baseurl }}/post_images/2015/08/shared_access_key_generator.png)

The SAS token has several parts. This is the string we will use to generate it: **SharedAccessSignature sig=%s&se=%i&skn=%s&sr=%s**. The first part is the signature, which we will create later. Then the expiration time, the key name used to generate the signature, and the URI encoded Event Hub URL.

The signature is generated by encrypting the combination of the URL and the expiration. It is encrypted using HMACSHA256. To do this we will create a function that takes the string to encrypt and the key and returns the encrypted string.

```f#
let encryptString (key : string) (message : string) =
 let keyBytes = System.Text.Encoding.UTF8.GetBytes(key)
 let messageBytes = System.Text.Encoding.UTF8.GetBytes(message)
 use hmac = new System.Security.Cryptography.HMACSHA256(keyBytes)
 let hashmessage = hmac.ComputeHash(messageBytes)
 Convert.ToBase64String(hashmessage)
 ```

 The expiration is in Unix time so we need to convert the DateTime. I am creating a token that lasts for 90 days, but this could be changed to accept a parameter.

 ```f#
 let toUnixTime (t : DateTime) =
    t.Subtract(new DateTime(1970, 1, 1)).TotalSeconds
 let expiry = DateTime.UtcNow.AddDays(90.0) |> toUnixTime |> Convert.ToInt32
 ```

 We now have enough to create the method that creates the token.

 ```f#
 let createToken (uri : string) keyName key =
 let toUnixTime (t : DateTime) =
    t.Subtract(new DateTime(1970, 1, 1)).TotalSeconds
 let expiry = DateTime.UtcNow.AddDays(90.0) |> toUnixTime |> Convert.ToInt32
 let encUri = System.Web.HttpUtility.UrlEncode(uri)
 let stringToSign = encUri + "\n" + expiry.ToString()
 let signature = encryptString key stringToSign |> System.Web.HttpUtility.UrlEncode
 sprintf "SharedAccessSignature sig=%s&se=%i&skn=%s&sr=%s" signature expiry keyName encUri
 ```

 Call this method and pass in your Event Hub URI, key name, and key. This will provide you with an SAS token that will expire after 90 days.
