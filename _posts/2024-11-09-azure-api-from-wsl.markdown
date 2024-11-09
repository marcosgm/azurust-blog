---
layout: post
title:  "First Azure API calls from WSL using Rust"
date:   2024-11-09 09:19:54 -0500
categories: azure wsl linux rust copilot blob storage generated
---
Once I've seen how to compile and debug Rust on Windows 11, adding a target to compile Linux code, now I'll connect to WSL and run my builds there on a pure Linux environment, and try my first API calls to Azure, in this case a Storage account.

We'll use GitHub Copilot to generate the code, and we'll learn how to fix its hallucinations due to outdated libraries by calling the raw HTTP APIs for Azure Blob Storage.

## Pre-requisites
- VSCode on Windows 11 capable of [WSL2](https://learn.microsoft.com/en-us/windows/wsl/install)
- Read the tutorial for [WSL and VScode](https://learn.microsoft.com/en-us/windows/wsl/tutorials/wsl-vscode)

## Launch VSCode with WSL remote

Open WSL, then click on the botton left icon >< and select "Connect to WSL". A refreshed window will say "Starting VS Code in WSL (UBuntu 24.04)".
![WSL Bar](/azurust-blog/assets/images/wslbar.png)

You can also see the content of your linux distribution by opening the Windows Terminal and opening a new tab in WSL - Ubuntu 24.04

Now install rust toolchains on Ubuntu (avoid using the APT route, using rustup is easier). Accept the defaults, it will install everything in your user environment, no root needed.
```
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

Then enable the VS Code Extension `rust-analyzer` and  `CodeLLDB` extensions, ensure you click on `Install in WSL: Ubuntu-24.04`. Enable the `Azure storage` extension too. Confirm that you can see them in the Extensions menu, WSL section:
![WSL Extensions](/azurust-blog/assets/images/wslextensionsazrust.png)

# Create fist Rust code

The goal of this code is to use the Azure Storage API from Rust. Let's start a new folder
```
mkdir azrust
cd azrust
cargo new azstorage-rust
code -a azstorage-rust
```

Now add a file in `src/` named `main.rs` and paste a helloworld example like
```rust
fn main() {
    println!("Hello, world!");
}
```
Click on the VSCode button to Run and Debug (Ctrl+Shift+D), accept the initial launch configuration prompt. Start the debugging (F5), you'll see "Hello world!" in the Terminal.
If rust-analyzer fails to load, restart the window.

# Create a Storage account

Using the Azure extensions, we'll create a storage account and get its storage key so we can later connect to it via API and upload a blob.

![AzLogin](/azurust-blog/assets/images/azfirstlogin.png) 

![New Storage account](/azurust-blog/assets/images/newstoracc.png)

Once asked, pick any region and give it a unique name

Now right-click on `Blob Containers` and select `Create Blob container`, name it `Blogpost`
![Blob Container](/azurust-blog/assets/images/blobcontainer.png)

Now right-click on the Blogpost container and select `Generate and copy SAS URL'
![SAS URL](/azurust-blog/assets/images/blobsasurl.png)

Save this URL in your notepad for the next step

## Generate Rust code to upload a file to Azure Storage using SAS URL

For this step, we're going to ask Copilot / ChatGPT / your preferred AI. Use this as a prompt: *Generate the required Rust code to connect to Azure Blog Storage container named "Blogpost" using a SAS URL, upload a text file named "example.txt" with the text "Hello World" in it* 

I use Github copilot, so I enabled the extension in WSL. Here's the Chat interface with the answer:
![Github copilot](/azurust-blog/assets/images/githubcopilotazstorage.png)

The generated code points us to the [Azure SDK for Rust](
https://github.com/Azure/azure-sdk-for-rust) which is under active development. The documentation pages are hosted in Docs.rs, filter by azure (https://docs.rs/releases/search?query=azure). There's a legacy crate, like this one: `https://docs.rs/azure_storage_blobs/latest/azure_storage_blobs/` and that's why Copilot/ChatGPT is picking it up. So the code above WILL NOT WORK in the near future, as it uses an unsupported SDK. The library version is also wrong.

Let's correct copilot and ask it to use the HTTP API calls to perform a Put Blob operation on the storage account. We add this prompt: *Do not use the azure_sdk_for_rust crate. Use the reqwest instead to call the Azure Storage Put Blob HTTP API*

We load the library by editing `Cargo.toml` with the required dependencies
```
[dependencies]
tokio = { version = "1", features = ["full"] }
reqwest = { version = "0.11", features = ["json", "multipart"] }
```

[Tokio](https://tokio.rs/) is an Asynchronous runtime framework for Rust, used when interacting with external APIs so our calls do not block execution and waste precious CPU time
[Reqwest](https://docs.rs/reqwest/latest/reqwest/) is a HTTP Client library for Rust. See examples in the [Cooking with Rust e-book](https://rust-lang-nursery.github.io/rust-cookbook/web/clients/apis.html)


In my ubuntu system, Cargo build failed due to lack of openssl libraries. Let's add them beforehand
```
sudo apt install libssl-dev openssl -y
```

The generated code forgot to import a trait in order to use write_all()
![reqwest write all](/azurust-blog/assets/images/reqwest_writeall.png)

It also forgot to add a mandatory HTTP header to indicate the Blob type, which we want it to be Block Blob. Read the [docs in this page](https://learn.microsoft.com/en-us/rest/api/storageservices/put-blob?tabs=shared-access-signatures)
```
Failed to upload file: "<?xml version=\"1.0\" encoding=\"utf-8\"?>\n<Error><Code>MissingRequiredHeader</Code><Message>An HTTP header that&apos;s mandatory for this request is not specified.\nRequestId:2ceb55b7-e01e-0008-68ef-32e2ef000000\nTime:2024-11-09T21:33:25.2107689Z</Message><HeaderName>x-ms-blob-type</HeaderName></Error>"
```

This is the correct code (make sure you replace the *SAS_URL* with yours)
```rust
use tokio::fs::File;
use tokio::io::AsyncReadExt;
use tokio::io::AsyncWriteExt;
use reqwest::Client;
use reqwest::header::{HeaderMap, HeaderValue, CONTENT_LENGTH, CONTENT_TYPE};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let sas_url = "REDACTED_PUT_YOUR_OWN_SAS_URL_HERE";
    let content = "Hello World";

    // Create the file and write content to it
    let blob_name = "example.txt";
    let mut file = File::create(blob_name).await?;
    file.write_all(content.as_bytes()).await?;

    // Read the file content into a buffer
    let mut file = File::open(blob_name).await?;
    let mut buffer = Vec::new();
    file.read_to_end(&mut buffer).await?;

    // Create the HTTP client
    let client = Client::new();

    // Create the request headers
    let mut headers = HeaderMap::new();
    headers.insert(CONTENT_TYPE, HeaderValue::from_static("text/plain"));
    headers.insert(CONTENT_LENGTH, HeaderValue::from(buffer.len()));
    headers.insert("x-ms-blob-type",HeaderValue::from_static("BlockBlob"));

    // Send the PUT request to upload the blob
    let response = client.put(sas_url)
        .headers(headers)
        .body(buffer)
        .send()
        .await?;

    if response.status().is_success() {
        println!("File uploaded successfully!");
    } else {
        println!("Failed to upload file: {:?}", response.text().await?);
    }

    Ok(())
}
```

Once you change the `Cargo.toml` and `src/main.rs` files with the above code, let's compile it and run it using the `Run and Debug (Ctrl+Shift+D)` button

It should produce a "File uploaded successfully!" message, and you should be able to see the file in your storage account. Double-click to see it's conent, "Hello World"
![first upload](/azurust-blog/assets/images/firstuploadedblob.png)

Now let's change the content in the code, re-compile and re-launch, and validate it overwrites the file.

![overwrite](/azurust-blog/assets/images/bloboverwrite.png)
Isn't this fun?