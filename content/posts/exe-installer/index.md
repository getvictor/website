+++
title = "How to create an EXE installer for your program"
description = "Quickly create an EXE installer for your program using the Inno Setup tool"
authors = ["Victor Lyuboslavsky"]
image = "exe-installer-headline.png"
date = 2024-09-11
categories = ["Software Development"]
tags = ["Installer", "Windows"]
draft = false
+++

## MSI versus EXE installers

When distributing software for Windows, you have two main options for installers: MSI and EXE. An MSI installer is a
Windows Installer package that contains installation information and files. It uses the Windows Installer service. On
the other hand, an EXE installer is a self-extracting executable file containing the installation files and an
installation program. EXE installers are more customizable and do not depend on the built-in Windows Installer
technology.

This article will show how to create an EXE installer for a program using the Inno Setup tool.

## Build your program

We will create a simple Hello World program using the Go programming language for this example.

With Go installed, we can build our program using the `go build` command. For example, given the source code in
`main.go`:

```go
package main

import "fmt"

func main() {
    fmt.Println("hello world")
}
```

We can build the program for Windows using:

```bash
GOOS=windows GOARCH=amd64 go build -o hello-world.exe main.go
```

## Download and install Inno Setup

We will need to use a Windows machine to create an EXE installer.

[Inno Setup](https://www.jrsoftware.org/isinfo.php) is a free installer for Windows programs. You can download it from
the [official website](https://www.jrsoftware.org/isdl.php). Once you have downloaded the installer, run it and follow
the installation instructions.

## Create an EXE installer

Launch the `Inno Setup Compiler` application. The main window will appear, with a toolbar and a script editor.

On the Welcome modal, choose `Create a new script file using the Script Wizard` and click `OK`.

{{< figure src="inno-setup-welcome.png" alt="Untitled Inno Setup Compile with Welcome modal open." >}}

Follow the instructions on several subsequent screens.

On the `Application Files` screen, add your program executable file and any other files, such as a README.

{{< figure src="inno-setup-application-files.png" alt="Application Files window of Inno Setup Script Wizard. hello-world.exe and hello-world.txt are specified." >}}

Continue following the instructions.

On the `Compiler Settings` screen, select the file name for your installer.

{{< figure src="inno-setup-compiler-settings.png" alt="Compiler Settings window of Inno Setup Script Wizard. hello-world-installer is specified as the compiler output base file name." >}}

Finally, click' Finish' after a couple more screens to generate the script.

Click `Yes` to compile the script.

Click `No` to save the script before compiling. If needed, it can be saved later.

{{< figure src="inno-setup-save-script.png" alt="Inno Setup modal asking: Would you like to save the script before compiling?" >}}

The Inno Setup Compiler will create an EXE installer for your program and put it in the `Documents/Output` folder.

You can try running the installer to make sure it works as expected.

## Further reading

- In the past, we demonstrated [how to code sign a Windows application](../code-signing-windows)
- Recently, we explained
  [how to measure Go test execution time and derive actionable insights](../go-test-execution-time)
- Previously, we wrote about [Go benchmarks](../optimizing-performance-of-go-app)

## Watch how to create an EXE installer

{{< youtube 1YeRYIhWqtA >}}

_Note:_ If you want to comment on this article, please do so on the YouTube video.
