+++
title = 'Inspecting keychain files on macOS'
description = 'Extracting info from macOS keychain files has gotten harder, but not impossible'
date = 2023-11-16
tags = ["macos", "cybersecurity"]
draft = false
+++

{{< youtube QBn_C2nl2ZE >}}

Keychains are the macOS’s method to track and protect secure information such as passwords, private keys, and certificates. Traditionally, the keychain information was stored in files, such as:

```
/Library/Keychains/System.keychain
/Library/Keychains/apsd.keychain
/System/Library/Keychains/SystemRootCertificates.keychain
/Users/<username>/Library/Keychains/login.keychain-db
```

In the last several years, Apple also introduced data protection keychains, such as the iCloud 
Keychain. Although the file-based keychains above are on the road to deprecation in favor of data 
protection keychains, current macOS systems still heavily rely on them. It is unclear when, if 
ever, these keychains will be replaced by data protection keychains.

Inspecting file-based keychains has gotten more difficult as Apple deprecated many of the APIs 
associated with them, such as [SecKeychainOpen](https://developer.apple.com/documentation/security/1396431-seckeychainopen).
In addition, excessive use of these deprecated APIs may result in corruption of the Login Keychain,
as mentioned in this [osquery issue](https://github.com/osquery/osquery/issues/7780).
By NOT using the deprecated APIs, the user only has access to the following keychains from the above list:

```
/Library/Keychains/System.keychain
/Users/<username>/Library/Keychains/login.keychain-db
```

Root certificates are missing. And the APSD (Apple Push Service Daemon) keychain is missing, which is used for device management, among other things.

So, how can app developers and IT professionals continue to have access to ALL of these keychain files?

One way is to continue using deprecated APIs until they stop working. We recommend making a secure copy of the keychain files before accessing them with the APIs.

Another option is to use the macOS [security](https://ss64.com/osx/security.html) command line tool. For example, to list root certificates, do the following:

```shell
sudo security find-certificate -a /System/Library/Keychains/SystemRootCertificates.keychain
```

A third, and hardest, option is to parse the [keychain files](https://github.com/libyal/dtformats/blob/main/documentation/MacOS%20keychain%20database%20file%20format.asciidoc) yourself. Some details on the keychain format are available. Please leave a comment if you or someone else has created a tool to parse Apple keychains.

The fourth option is to use an existing tool, such as [osquery](https://www.osquery.io/). Osquery is an open-source tool built for security and IT professionals. Osquery developers are working on fixing any issues to continue providing access to macOS keychain files via the following tables:

 - [certificates](https://fleetdm.com/tables/certificates)
 - [keychain_acls](https://fleetdm.com/tables/keychain_acls)
 - [keychain_items](https://fleetdm.com/tables/keychain_items)
