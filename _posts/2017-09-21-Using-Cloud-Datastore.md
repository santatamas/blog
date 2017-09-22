---
layout: post
title: Using Cloud Datastore with dotnetcore
date:   2017-09-21 21:57:00 +0100
categories: Blu Docker dotnetcore
---

As part of my ongoing investigation into possible DB technologies, I've decided to give Cloud Datastore a try.
Cloud Datastore is a managed NoSql database service provided by Google, as part of GCloud. Nowadays, all the biggest players has came up with their own NoSql alternatives (DynamoDB for AWS, Cosmos for Azure, etc.), and in my opinion their biggest selling point is their price, and availability. My current domain objects can be modelled as documents anyway, so...happy times!

Based on [Google's price calculator](https://cloud.google.com/products/calculator/), the DB will cost **$0**. I'm lucky, because that's the exact number I budgeted for it.

In my experience so far (I've only worked with DynamoDB before), the main hassle with managed services is mocking them out in the local dev environment. Once your application is deployed, you won't really have any issues as the official client libraries normally take care of the connection and authentication parts.

Both AWS and Google provides slimmed down emulators for their services. In case of DynamoDB it's a runnable jar file. Google decided to bundle their emulators (plural, because they have emulators for literally everything) into the gcloud SDK.


[Running the Cloud Datastore Emulator](https://cloud.google.com/datastore/docs/tools/datastore-emulator)


[GCloud SDK](https://cloud.google.com/sdk/docs/quickstart-mac-os-x)

# Setting up the local developer environment

Let's start by spinning up the latest google SDK in a container. Oh, forgot to mention...you'll need Docker for this!

```shell
docker run --rm -e CLOUDSDK_CORE_PROJECT=aaaaaaa-aaaaa-111111 -p 8081:8081 google/cloud-sdk:latest
/usr/lib/google-cloud-sdk/platform/cloud-datastore-emulator/cloud_datastore_emulator start
--host=0.0.0.0 --port=8081 --testing
```

But wait, what do we have here? The `docker run` should be familiar to everyone. We have to provide the `CLOUDSDK_CORE_PROJECT` environment variable to the container, because *reasons*. You can leave it as it is, the server doesn't actually *need* a project ID. But it won't start without it. Go figure.

Then we publish the 8081 port to the host, so our sample app will be able to connect to the container. Then we specify the image. The latest image contains all the SDK components pre-installed, so if you fancy a slimmed-down version, you should build your own.

The last bit points to the emulator binary, and tells it to start up and listen on the `0.0.0.0` address. By default it's listening on *localhost*, and that makes it impossible to access it from outside the container. Hence the overcomplicated startup parameters. I won't even tell you how much time I spent on figuring this one out :)

Once it's up and running, you will see something like this on the output:
```shell
API endpoint: http://0.0.0.0:8081
If you are using a library that supports the DATASTORE_EMULATOR_HOST environment variable, run:

  export DATASTORE_EMULATOR_HOST=0.0.0.0:8081

Dev App Server is now running.
```

A quick `curl http://localhost:8081` should result in an **Ok** response. Our dev server is up and running!

# Accessing DB from dotnetcore

Let's move on to the dotnet part, by creating a simple console application with `dotnet new console`.

Add the following NuGet package reference to the .csproj:
```xml
<ItemGroup>
  <PackageReference Include="Google.Cloud.Datastore.V1" Version="2.0.0" />
</ItemGroup>
```

To connect to the *local* DB, we're going to use the official sample class. The only thing I had to change is the `DatastoreClient` initialisation, as the default constructor will use the real service.

**Don't believe the emulator's lies!**
Setting the `DATASTORE_EMULATOR_HOST=0.0.0.0:8081`environment variable does **NOT** work with the dotnetcore client. Again, because *reasons*.

Below you can find the full class:
```c#
static void Main(string[] args)
{
    // Create a custom Datastore Client pointing to our dev container
    var channel = new Channel("127.0.0.1", 8081, ChannelCredentials.Insecure);
    DatastoreClient client = DatastoreClient.Create(channel);

    // Instantiates a datastore instance with the custom client
    DatastoreDb db = DatastoreDb.Create("aaaaaaa-aaaaa-000000", "", client);

    // Uncomment this if you want to use the default (live) datastore
    //DatastoreDb db = DatastoreDb.Create("aaaaaaa-aaaaa-000000");

    string kind = "Task";
    string name = "sampletask1";

    KeyFactory keyFactory = db.CreateKeyFactory(kind);

    // The Cloud Datastore key for the new entity
    Key key = keyFactory.CreateKey(name);

    var task = new Entity { Key = key, ["description"] = "Buy milk" };
    using (DatastoreTransaction transaction = db.BeginTransaction())
    {
        transaction.Upsert(task);
        transaction.Commit();

        Console.WriteLine($"Saved {task.Key.Path[0].Name}: {(string)task["description"]}");
    }
}
```

We're almost set. If you try to run it now, it'll complain about missing credentials. Weird stuff, because I'd expect it to just work locally with a custom client, but nah...
You'll have to get your credentials from the [Google Cloud Console](https://console.cloud.google.com/apis/credentials)


Click `Create credentials -> Service account key`

This should download your creds in JSON.
Save it to somewhere safe, and add the following environment variable to the project's startup parameters:
`GOOGLE_APPLICATION_CREDENTIALS={PATH_TO_YOUR_CREDS_FILE}`

Now we're set! Run your project, and if everything's working correctly, you'll see the following output:

```shell
Saved sampletask1: Buy milk
```

That's it!