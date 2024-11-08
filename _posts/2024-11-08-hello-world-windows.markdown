---
layout: post
title:  "Simple Rust Hello World in Windows 11"
date:   2024-11-08 09:19:54 -0500
categories: azure rust blog about
---
Here's what you need to do on a Windows 11 laptop to run Rust
For complete info on installing Rust on Windows, read [the official doc](https://learn.microsoft.com/en-us/windows/dev-environment/rust/setup) or [this alternative blogpost](https://www.petergirnus.com/blog/how-to-install-rust-on-windows)

## Pre-requisites
- Winget
- windows Terminal

To be installed via winget
- MS C++ Build tools
- VSCode
- Git
- Rust build tools

## Step-by-step Rust Installation

Start by opening the Terminal, and install VSCode, git and BuildTools
```
winget install Microsoft.VisualStudio.2022.Community --silent --override "--wait --quiet --add ProductLang En-us --add Microsoft.VisualStudio.Workload.NativeDesktop --includeRecommended"

winget install -e Microsoft.VisualStudioCode Microsoft.Git Microsoft.VisualStudio.2022.BuildTools
```

Now install  Rust-up, a tool that will install all pre-requisites and system settings on Windows (it will take a couple minutes)

```
winget install Rustlang.Rustup
```
![Image](/images/wingetrustup.png)


Now open VS Code, go the Extensions menu and install the *rust-analyzer* and the *CodeLLDB* extensions

![Image](/images/vscoderustextensions.png)

Now add the rust binary needed for those extensions using *rustup"
```
rustup component add rust-analyzer
```

## Hello World

Open your windows terminal, create a folder for this tutorial, clone it with git and open it in VSCode

```
mkdir azurust
cd azurust
git clone https://github.com/meta-rust/rust-hello-world.git
code rust-hellow-world
```

Open the VSCode Terminal and validate that Rust and the Cargo tool works
```
rust -version
cargo --version
```

Open the file src/main.rs, then launch the Run button in VSCode or by pressing F5

When you run an app under the CodeLLDB extension and debugger for the first time, you'll see a dialog box saying "Cannot start debugging because no launch configuration has been provided". Click OK to see a second dialog box saying "Cargo.toml has been detected in this workspace. Would you like to generate launch configurations for its targets?". Click Yes. Then close the launch.json file and begin debugging again.

## Debugging basics

Now let's add a second message in the screen, and learn how to debug the code

```
fn main() {
    println!("Hello, world!");
    println!("And hello again!")
}
```

In VS Code, insert a breakpoint between the two Println lines, then run with Debug enabled

![Image](/images/debughelloworld.png)

