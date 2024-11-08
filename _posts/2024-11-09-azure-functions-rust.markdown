---
layout: post
title:  "First Azure Function Hello World using Azure Functions"
date:   2024-11-09 00:19:54 -0500
categories: azure functions rust
---
This post shows how to run a simple Rust Hello world example on an HTTP Endpoint hosted with an Azure Functions runtime. Here's the [Microsoft documentation about it](https://learn.microsoft.com/en-us/azure/azure-functions/create-first-function-vs-code-other?tabs=rust%2Cwindows)

As there's no Azure SDK For Rust yet, we'll launch Azure resources using Bicep

## Pre-requisites
- Azure CLI with Bicep
- An active Azure subscription, authenticated with `az login` , see [info in this page](https://learn.microsoft.com/en-us/cli/azure/get-started-with-azure-cli)

## Step by step

Install Azure Functions Core Tools and latest versions of the Az CLI and Bicep
```
winget install Microsoft.Azure.FunctionsCoreTools Microsoft.Bicep Microsoft.AzureCLI
```

Let's start by creating a folder for this test
```
mkdir func-rust-helloworld
cd func-rust-helloworld
code .
```


## Initialize the workspace folder that will contain the Function

Start by installing the `Azure Functions` extension for VSCode
![Image](/azurust-blog/assets/images/functionsextension.png)

- Now go to the Azure extension, Workspace, then click on `Create Function Project`, select the current folder
- Then, when it asks for "Select a Language", pick `Custom Handler`, as currently there's no Rust starter template. 
- Next, it'll ask "Select a template for your project's first function", pick `HTTP Trigger`, and accept the name "HttpTrigger1". 
- Select Authorization Level `Anonymous`


![Image](/azurust-blog/assets/images/initfuncworkspace.png) 

Now let's create the Rust code that will responde to the HTTP requests
```
cargo init --name handler
```

We'll use Warp to handle the requests, so open `Cargo.toml` and add these dependencies
```
[dependencies]
warp = "0.3"
tokio = { version = "1", features = ["rt", "macros", "rt-multi-thread"] }
```

Now edit `src/main.rs`

```ruby
use std::collections::HashMap;
use std::env;
use std::net::Ipv4Addr;
use warp::{http::Response, Filter};

#[tokio::main]
async fn main() {
    let example1 = warp::get()
        .and(warp::path("api"))
        .and(warp::path("HttpExample"))
        .and(warp::query::<HashMap<String, String>>())
        .map(|p: HashMap<String, String>| match p.get("name") {
            Some(name) => Response::builder().body(format!("Hello, {}. This HTTP triggered function executed successfully.", name)),
            None => Response::builder().body(String::from("This HTTP triggered function executed successfully. Pass a name in the query string for a personalized response.")),
        });

    let port_key = "FUNCTIONS_CUSTOMHANDLER_PORT";
    let port: u16 = match env::var(port_key) {
        Ok(val) => val.parse().expect("Custom Handler port is not a number!"),
        Err(_) => 3000,
    };

    warp::serve(example1).run((Ipv4Addr::LOCALHOST, port)).await
}
```

Now compile and let cargo install the dependent libraries
```
cargo build --release
cp target/release/handler.exe .
```

## Configure Azure Functions to execute the HTTP handler

The function host needs to be configured to run your custom handler binary when it starts Open `host.json`, and replace the "customHandler" section with the following:
```
"customHandler": {
  "description": {
    "defaultExecutablePath": "handler.exe",
    "workingDirectory": "",
    "arguments": []
  },
  "enableForwardingHttpRequest": true
}
```

You can now test your function locally thanks to the Azure Function's local runtime that was installed via the VSCode extension.
```
func start
```

Open this URL: `http://localhost:7071/api/HttpTrigger1?name=AzuRust`

## Deploy the function to Azure

Let's login to Azure from VSCode

![Image](/azurust-blog/assets/images/azfirstlogin.png) 

And create a new Function App. By default, for Custom runtimes, it will be a Windows hosting plan.
![Image](/azurust-blog/assets/images/createfuncapp.png)  

Enter a globally unique name, like *helloworld2311* , with Custom Handler, any region (try again if the region fails due to quota)

Now proceed to deploy from the Workspace shortcut "Deploy to Azure", select your subscription and your function app.
![Image](/azurust-blog/assets/images/deployfuncazure.png)  

You can connect to the Log Stream once the function has been properly deployed so you can see your event logs.
![Image](/azurust-blog/assets/images/funcstreamlogs.png)  

Get the public function URL from the Function App menu
![Image](/azurust-blog/assets/images/getfuncurl.png)  

Example public URL with parameters
`https://helloworld2311.azurewebsites.net/api/HttpTrigger1?name=myfirstfunction`

## Deploying to a Linux runtime from a Windows laptop

If you selected an Azure Function Apps runtime on Linux, remember we compiled an EXE file for Windows. You can perform the following changes to add a build target of type `x86_64-unknown-linux-musl`
https://learn.microsoft.com/en-us/azure/azure-functions/create-first-function-vs-code-other?tabs=rust%2Cwindows#compile-the-custom-handler-for-azure

Once you change your host.json `defaultExecutablePath` you can redeploy to a Linux runtime from a Windows workstation