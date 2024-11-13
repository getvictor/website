+++
title = "2 ways to turn a script into a macOS install package"
description = "How to create a script-only macOS install package"
authors = ["Victor Lyuboslavsky"]
image = "script-package-headline.png"
date = 2024-11-13
categories = ["DevOps & Infrastructure"]
tags = ["macOS", "Installer"]
draft = false
+++

- [Create a script-only install package using the `pkgbuild` command](#create-a-script-only-install-package-using-the-pkgbuild-command)
- [Create a script-only install package using the Packages app](#create-a-script-only-install-package-using-the-packages-app)

## What is a macOS install package?

A macOS install package is a file that contains the files and scripts needed to install an application on a macOS
system. It is commonly used to distribute software to macOS users and can contain multiple files, scripts, and metadata.

## Why create a macOS install package that only runs a script?

Sometimes, you must distribute a script that performs a specific task on a macOS system, such as fixing a known issue.
You can create a macOS install package that contains the script and any other files needed to run that script. This
workflow allows you to distribute the script as an install package that users can easily install on their macOS systems.

Another reason to create a macOS install package that only runs a script is to use a third-party installer instead of
the built-in macOS installer. This custom installer can provide additional features and customization options.

## Create a script-only install package using the `pkgbuild` command

The `pkgbuild` command is a command-line tool included with macOS. It allows you to create macOS install packages from
the command line.

First, create a directory for your script files:

```bash
mkdir Scripts
```

Create a script file in the above directory called `postinstall` that contains the script you want to run. Below is an
example script for testing:

```bash
#!/bin/bash

# installer script variables:
# $0 = path to the script
# $1 = path to the package
# $2 = target location, i.e., /Applications
# $3 = target volume, i.e., /Volumes/Macintosh HD
# $4 = "/" if this is the startup disk

mkdir -p /opt/hello
target="/opt/hello/hello.txt"
echo "\$0=$0" > $target
echo "\$1=$1" >> $target
echo "\$2=$2" >> $target
echo "\$3=$3" >> $target
echo "\$4=$4" >> $target
echo "\$INSTALL_PKG_SESSION_ID=$INSTALL_PKG_SESSION_ID" >> $target
echo "\$USER=$USER" >> $target
echo "\$HOME=$HOME" >> $target

# Always succeed
exit 0
```

The above script creates a directory `/opt/hello` and writes various script variables to a file `/opt/hello/hello.txt`.

Make sure the script is executable:

```bash
chmod +x Scripts/postinstall
```

Create the installation package using the `pkgbuild` command:

```bash
pkgbuild --nopayload --scripts Scripts --identifier com.victoronsoftware.pkgbuild-demo --version 1.0 PkgbuildDemo.pkg
```

The above command creates an install package `PkgbuildDemo.pkg`.

The `--nopayload` flag tells `pkgbuild` that there are no application files to include in the package. The `--scripts`
flag specifies the directory containing the scripts to run during the installation. The scripts directory may also
contain additional files needed by the script.

At this point, you can try installing the package on a test macOS system:

```bash
sudo installer -pkg PkgbuildDemo.pkg -target /
```

## Create a script-only install package using the Packages app

One popular GUI tool for creating macOS installer packages is the
[Packages app](http://s.sudre.free.fr/Software/Packages/about.html).

Download and install the Packages app.

Create a new project in the Packages app using the Raw Package template.

{{< figure src="packages-app-new-project.png" alt="Choose a template for your project. A Raw Package project lets you install files at specific locations." >}}

Choose the name and location of your project.

In the Scripts tab, choose a Post-installation script. Add additional script resource files if needed for the script.

{{< figure src="packages-app-add-script.png" alt="Scripts tab is selected on the top. From the two options, pre-installation and post-installation, the post-installation contains an exec file." >}}

Save your project with **File > Save** and build the package with **Build > Build**.

The tool will save the new PKG file to your project directory.

## Analyze the install package

To analyze the install package, you can use a tool like
[Suspicious Package](https://www.mothersruin.com/software/SuspiciousPackage/get.html)

{{< figure src="suspicious-package.png" alt="Suspicious Package app showing the contents of the postinstall script." >}}

## Sign and notarize the install package

Before distributing the package to users, you may need to sign and notarize it.

To sign the package, you need a Developer ID Installer certificate. The Apple Developer Program currently costs 99 USD
per membership year. To sign your package, place the certificate and corresponding private key (together called an
"identity") into your keychain. Then, you can sign the package using the `productsign` command-line utility:

```bash
productsign --sign "Developer ID Installer: ********" ~/PkgbuildDemo.pkg ~/PkgbuildDemo-signed.pkg
```

You can notarize your package with Apple using the `notarytool` command-line utility. For more information, see
[Notarizing macOS software before distribution](https://developer.apple.com/documentation/security/notarizing-macos-software-before-distribution?language=objc).

## Distribute the install package

You can distribute the package by posting a download link on your website, through a package manager, or using your MDM
tool.

If you're using a macOS MDM platform such as [Fleet](https://fleetdm.com/device-management), you can upload the package
to the MDM and deploy it to your managed devices.

## Further reading

- In the past, we showed [how to create an EXE installer for Windows](../exe-installer/) and
  [code sign a Windows application](../code-signing-windows/).
- We also covered [using Mutual TLS (mTLS) with macOS keychain](../mtls-with-apple-keychain/).

## Watch how to create a script-only macOS install package

{{< youtube _NWQS0Eu74k >}}

_Note:_ If you want to comment on this article, please do so on the YouTube video.
