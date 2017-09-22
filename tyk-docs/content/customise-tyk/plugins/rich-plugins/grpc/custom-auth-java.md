---
date: 2017-03-24T13:28:45Z
title: Create Custom Authentication Plugin with Java
menu:
  main:
    parent: "gRPC"
weight: 3 
---

## <a name="introduction"></a>Introduction

This tutorial will guide you through the creation of a gRPC-based Java plugin for Tyk.
Our plugin will inject a header into the request before it gets proxied upstream.

For additional information about gRPC, check the official documentation [here](https://grpc.io/docs/guides/index.html).

## <a name="Requirements"></a>Requirements

* Tyk Gateway: This can be installed using standard package management tools like Yum or APT, or from source code. See [here][1] for more installation options.
* The Tyk CLI utility, which is bundled with our RPM and DEB packages, and can be installed separately from [https://github.com/TykTechnologies/tyk-cli][2].
* Gradle Build Tool: https://gradle.org/install/.
* gRPC tools: https://grpc.io/docs/quickstart/csharp.html#generate-grpc-code
* Java JDK 7 or higher.


## <a name="what-is-grpc"></a>What is gRPC?

gRPC is a very powerful framework for RPC communication across different languages. It was created by Google and makes heavy use of HTTP2 capabilities and the Protocol Buffers serialization mechanism.

## <a name="why-use-it"></a>Why Use it for Plugins?
When it comes to built-in plugins, we have been able to integrate several languages like Python, Javascript & Lua in a native way: this means the middleware you write using any of these languages runs in the same process. For supporting additional languages we have decided to integrate gRPC connections and perform the middleware operations outside of the Tyk process. The flow of this approach is as follows: 

* Tyk receives a HTTP request.
* Your gRPC server performs the middleware operations (for example, any modification of the request object).
* Your gRPC server sends the request back to Tyk.
* Tyk proxies the request to your upstream API.

The sample code that we'll use implements a very simple authentication layer using Java and the proper gRPC bindings generated from our Protocol Buffers definition files.

## <a name="create"></a>Create the Plugin

### Setting up the Java Project

We will use the Gradle build tool to generate the initial files for our project:

```{.copyWrapper}
cd ~
mkdir tyk-plugin
cd tyk-plugin
gradle init
```

We now have a `tyk-plugin` directory containing the basic skeleton of our application.

We now need to edit the following file:
`/tyk-plugin/build.gradle`

Add the following to `build.gradle`

```{.copyWrapper}
buildscript {
   repositories {
       mavenCentral()
   }
   dependencies {
       classpath 'com.google.protobuf:protobuf-gradle-plugin:0.8.1'
   }
}

plugins {
 id "com.google.protobuf" version "0.8.1"
 id "java"
 id "application"
 id "idea"
}

protobuf {
   protoc {
       artifact = "com.google.protobuf:protoc:3.3.0"
   }
   plugins {
       grpc {
           artifact = 'io.grpc:protoc-gen-grpc-java:1.5.0'
       }
   }
   generateProtoTasks {
       all()*.plugins {
           grpc {}
       }
   }
   generatedFilesBaseDir = "$projectDir/src/generated"
}

sourceCompatibility = 1.8
targetCompatibility = 1.8

repositories {
   mavenCentral()
}

dependencies {
   compile 'io.grpc:grpc-all:1.5.0'
}

idea {
   module {
       sourceDirs += file("${projectDir}/src/generated/main/java");
       sourceDirs += file("${projectDir}/src/generated/main/grpc");
   }
}

task runServer(type: JavaExec) {
   classpath = sourceSets.main.runtimeClasspath
   main = 'com.testorg.testplugin.PluginServer'
}

startScripts.enabled = false

task pluginServer(type: CreateStartScripts) {
   mainClassName = "com.testorg.testplugin.PluginServer"
   applicationName = "plugin-server"
   outputDir = new File(project.buildDir, 'tmp')
   classpath = jar.outputs.files + project.configurations.runtime
}

applicationDistribution.into("bin") {
   from(pluginServer)
   fileMode = 0755
}
```



### Create the Directory for the Server Class

```{.copyWrapper}
cd ~/tyk-plugin
mkdir -p src/main/java/com/testorg/testplugin
```

### gRPC tools and bindings generation

We need to download the Tyk Protocol Buffers definition files, these files contains the data structures used by Tyk. See [Data Structures](/docs/customise-tyk/plugins/rich-plugins/rich-plugins-data-structures/) for more information:

```{.copyWrapper}
cd ~/tyk-plugin
git clone https://github.com/TykTechnologies/tyk-protobuf
mv tyk-protobuf/proto src/main/proto
```


To generate the Protocol Buffers bindings we use the Gradle build task:

```{.copyWrapper}
gradle build
```

If you need to customize any setting related to the bindings generation step, check the `build.gradle` file.



### Server Implementation

We need to implement two classes: one class will contain the request dispatcher logic and the actual middleware implementation. The other one will implement the gRPC server using our own dispatcher.

From the `/tyk-plugin/src/main/java/com/testorg/testplugin` directory, create a file named `PluginDispatcher.java` with the following code:

```{java}
package com.testorg.testplugin;

import coprocess.DispatcherGrpc;
import coprocess.CoprocessObject;

public class PluginDispatcher extends DispatcherGrpc.DispatcherImplBase {

    @Override
    public void dispatch(CoprocessObject.Object request,
            io.grpc.stub.StreamObserver<CoprocessObject.Object> responseObserver) {
        CoprocessObject.Object modifiedRequest = null;

        switch (request.getHookName()) {
            case "MyPreMiddleware":
                modifiedRequest = MyPreHook(request);
            default:
            // Do nothing, the hook name isn't implemented!
        }

        // Return the modified request (if the transformation was done):
        if (modifiedRequest != null) {
            responseObserver.onNext(modifiedRequest);
        };

        responseObserver.onCompleted();
    }

    CoprocessObject.Object MyPreHook(CoprocessObject.Object request) {
        CoprocessObject.Object.Builder builder = request.toBuilder();
        builder.getRequestBuilder().putSetHeaders("customheader", "customvalue");
        return builder.build();
    }
}
```

In the same directory, create a file named `PluginServer.java` with the following code. This is the server implementation:

```{java}
package com.testorg.testplugin;

import coprocess.DispatcherGrpc;

import io.grpc.Server;
import io.grpc.ServerBuilder;
import io.grpc.stub.StreamObserver;
import java.io.IOException;
import java.util.logging.Level;
import java.util.logging.Logger;

public class PluginServer {

    private static final Logger logger = Logger.getLogger(PluginServer.class.getName());
    static Server server;
    static int port = 5555;

    public static void main(String[] args) throws IOException, InterruptedException {
        System.out.println("Initializing gRPC server.");

        // Our dispatcher is instantiated and attached to the server:
        server = ServerBuilder.forPort(port)
                .addService(new PluginDispatcher())
                .build()
                .start();

        blockUntilShutdown();

    }

    static void blockUntilShutdown() throws InterruptedException {
        if (server != null) {
            server.awaitTermination();
        }
    }
}
```

To run the gRPC server we can use the following command:

```{.copyWrapper}
cd ~/tyk-plugin
gradle runServer
```

The gRPC server will listen on port 5555 (as defined in `Server.java`). In the next steps we'll setup the plugin bundle and modify Tyk to connect to our gRPC server.


## <a name="bundle"></a>Setting up the Plugin Bundle

We need to create a manifest file within the `tyk-plugin` directory. This file contains information about our plugin and how we expect it to interact with the API that will load it. This file should be named `manifest.json` and needs to contain the following:

```{json}
{
    "custom_middleware": {
        "driver": "grpc",
        "pre": [{
            "name": "MyPreMiddleware"
        }]
    }
}
```

* The `custom_middleware` block contains the middleware settings like the plugin driver we want to use (`driver`) and the hooks that our plugin will expose. We use the `auth_check` hook for this tutorial. For other hooks see [here](https://tyk.io/docs/customise-tyk/plugins/rich-plugins/rich-plugins-work/#coprocess-dispatcher-hooks).
* The `name` field references the name of the function that we implement in our plugin code - `MyAuthMiddleware`. This will be handled by our dispatcher gRPC method, implemented in `PluginServer.java`.


To bundle our plugin run the following command in the `tyk-plugin` directory. Check your tyk-cli install path first:

```{.copyWrapper}
/opt/tyk-gateway/utils/tyk-cli bundle build -y
```


A plugin bundle is a packaged version of the plugin. It may also contain a cryptographic signature of its contents. The `-y` flag tells the Tyk CLI tool to skip the signing process in order to simplify the flow of this tutorial. 

For more information on the Tyk CLI tool, see [here](https://tyk.io/docs/customise-tyk/plugins/rich-plugins/plugin-bundles/#using-the-bundler-tool).

You should now have a `bundle.zip` file in the `tyk-plugin` directory.

## <a name="publish"></a>Publish the Plugin

To publish the plugin, copy or upload `bundle.zip` to a local web server like Nginx, or Apache or storage like Amazon S3. For this tutorial we'll assume you have a web server listening on `localhost` and accessible through `http://localhost`.

## <a name="configure-tyk"></a>Configure Tyk

You will need to modify the Tyk global configuration file `tyk.conf` to use gRPC plugins. The following block should be present in this file:

```{.copyWrapper}
"coprocess_options": {
    "enable_coprocess": true,
    "coprocess_grpc_server": "tcp://localhost:5555"
},
"enable_bundle_downloader": true,
"bundle_base_url": "http://localhost/bundles/",
"public_key_path": ""
```


### tyk.conf Options

* `enable_coprocess`: This enables the plugin.
* `coprocess_grpc_server`: This is the URL of our gRPC server.
* `enable_bundle_downloader`: This enables the bundle downloader.
* `bundle_base_url`: This is a base URL that will be used to download the bundle. You should replace the bundle_base_url with the appropriate URL of the web server that's serving your plugin bundles. For now HTTP and HTTPS are supported but we plan to add more options in the future (like pulling directly from S3 buckets).
* `public_key_path`: Modify `public_key_path` in case you want to enforce the cryptographic check of the plugin bundle signatures. If the `public_key_path` isn't set, the verification process will be skipped and unsigned plugin bundles will be loaded normally.


### Configure an API Definition

There are two important parameters that we need to add or modify in the API definition.
The first one is `custom_middleware_bundle` which must match the name of the plugin bundle file. If we keep this with the default name that the Tyk CLI tool uses, it will be `bundle.zip`:

```{json}
"custom_middleware_bundle": "bundle.zip"
```

Assuming the `bundle_base_url` is `http://localhost/bundles/`, Tyk will use the following URL to download our file:

`http://localhost/bundles/bundle.zip`

The second parameter is specific to this tutorial, and should be used in combination with `use_keyless` to allow an API to authenticate against our plugin:

```{json}
"use_keyless": false,
"enable_coprocess_auth": true
```


`enable_coprocess_auth` will instruct the Tyk gateway to authenticate this API using the associated custom authentication function that's implemented by our plugin.

### Configuration via the Tyk Dashboard

To attach the plugin to an API, from the **Advanced Options** tab in the **API Designer** enter `bundle.zip` in the **Plugin Bundle ID** field.

![Plugin Options][3]

We also need to modify the authentication mechanism that's used by the API.
From the **Core Settings** tab in the **API Designer** select **Use Custom Auth (plugin)** from the **Target Details - Authentication Mode** drop-down list. 

![Advanced Options][4]

## <a name="testing"></a>Testing the Plugin


At this point we have our test HTTP server ready to serve the plugin bundle and the configuration with all the required parameters. To run the gRPC server use:

```{.copyWrapper}
gradle runServer
```


The final step is to start or restart the **Tyk Gateway** (this may vary depending on how you set up Tyk):

```{.copyWrapper}
service tyk-gateway start
```


A simple CURL request will be enough for testing our custom authentication middleware.

This request will trigger an authentication error:

```{.copyWrapper}
curl http://localhost:8080/my-api/my-path -H 'Authorization: badtoken'
```

This will trigger a successful authentication. We're using the token that's specified in our server implementation:


```{.copyWrapper}
curl http://localhost:8080/my-api/my-path -H 'Authorization: abc123'
```

## <a name="next"></a>What's Next?

In this tutorial we learned how Tyk gRPC plugins work. For a production-level setup we suggest the following:

* Configure an appropriate web server and path to serve your plugin bundles.











[1]: https://tyk.io/docs/get-started/with-tyk-on-premise/installation/
[2]: https://github.com/TykTechnologies/tyk-cli
[3]: /docs/img/dashboard/system-management/plugin_options.png
[4]: /docs/img/dashboard/system-management/plugin_auth_mode.png





