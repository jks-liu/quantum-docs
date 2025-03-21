---
author: bradben
description: Describes how to update your Q# programming language projects and the Quantum Development Kit (QDK) to the current version.
ms.author: brbenefield
ms.date: 03/30/2022
ms.service: azure-quantum
ms.subservice: qdk
ms.topic: how-to
no-loc: ['Q#', '$$v', Quantum Development Kit, target, targets]
title: Update the Quantum Development Kit (QDK)
uid: microsoft.quantum.update-qdk
---

# Update the Quantum Development Kit (QDK) to the latest version

Learn how to update the Quantum Development Kit (QDK) to the latest version.

This article assumes that you already have the QDK installed. If you are installing for the first time, then please refer to the [installation guide](xref:microsoft.quantum.install-qdk.overview).

We recommend keeping up to date with the latest QDK release. Follow this update guide to upgrade to the most recent QDK version. 
The process consists of two parts:

1. Updating your existing Q# files and projects to align your code with any updated syntax.
2. Updating the QDK itself for your chosen development environment.

## Update Q# projects 

Regardless of whether you are using C# or Python to host Q# operations, follow these instructions to update your Q# projects.

1. First, check that you have the latest version of the [.NET SDK 6.0](https://dotnet.microsoft.com/download). Run the following command in the command prompt:

    ```dotnetcli
    dotnet --version
    ```

    Verify the output is `6.0.100` or higher. If not, install the [latest version](https://dotnet.microsoft.com/download) and check again. Then follow the instructions below depending on your setup (Visual Studio, Visual Studio Code, or directly from the command prompt).

### Update Q# projects in Visual Studio
 
1. Update to the latest version of Visual Studio 2022, see [here](/visualstudio/install/update-visual-studio) for instructions.
2. Open your solution in Visual Studio.
3. From the menu, select **Build** -> **Clean Solution**.
4. In each of your .csproj files, update the target framework to `net6.0` (or `netstandard2.1` if it is a library project).
    That is, edit lines of the form:

    ```xml
    <TargetFramework>net6.0</TargetFramework>
    ```

    You can find more details on specifying target frameworks [here](/dotnet/standard/frameworks#how-to-specify-target-frameworks).

5. In each of the .csproj files, set the SDK to `Microsoft.Quantum.Sdk`, as indicated in the line below. Please notice that the version number should be the latest available, and you can determine it by reviewing the [release notes](xref:microsoft.quantum.relnotes-qdk).

    ```xml
    <Project Sdk="Microsoft.Quantum.Sdk/0.24.201332">
    ```

6. Save and close all files in your solution.

7. Select **Tools** -> **Command Line** -> **Developer Command Prompt**. Alternatively, you can use the package management console in Visual Studio.

8. For each project in the solution, run the following command to **remove** this package:

    ```dotnetcli
    dotnet remove [project_name].csproj package Microsoft.Quantum.Development.Kit
    ```

   If your projects use any other Microsoft.Quantum or Microsoft.Azure.Quantum packages (for example, Microsoft.Quantum.Numerics), run the `add` command for these to update the version used.

    ```dotnetcli
    dotnet add [project_name].csproj package [package_name]
    ```

9. Close the command prompt and select **Build** -> **Build Solution** (do *not* select Rebuild Solution).

You can now skip ahead to [update your Visual Studio QDK extension](#update-the-qdk-for-visual-studio-extension).


### Update Q# projects in Visual Studio Code

1. In Visual Studio Code, open the folder containing the project to update.
2. Select **Terminal** -> **New Terminal**.
3. Follow the instructions for updating using the command prompt (directly below).

### Update Q# projects using the command prompt

1. Navigate to the folder containing your main project file.

2. Run the following command:

    ```dotnetcli
    dotnet clean [project_name].csproj
    ```

3. Determine the current version of the QDK. To find it, you can review the [release notes](xref:microsoft.quantum.relnotes-qdk). The version will be in a format similar to `0.24.201332`.

4. In each of your `.csproj` files, go through the following steps:

    - Update the target framework to `net6.0` (or `netstandard2.1` if it is a library project). That is, edit lines of the form:

        ```xml
        <TargetFramework>net6.0</TargetFramework>
        ```

        You can find more details on specifying target frameworks [here](/dotnet/standard/frameworks#how-to-specify-target-frameworks).

    - Replace the reference to the SDK in the project definition. Make sure that the version number corresponds to the value determined in **step 3**.

        ```xml
        <Project Sdk="Microsoft.Quantum.Sdk/0.24.201332">
        ```

    - Remove the reference to package `Microsoft.Quantum.Development.Kit` if present, which will be specified in the following entry:

        ```xml
        <PackageReference Include="Microsoft.Quantum.Development.Kit" Version="0.24.201332" />
        ```

    - Update the version of the all the Microsoft Quantum packages to the most recently released version of the QDK (determined in **step 3**). Those packages are named with the following patterns:

        ```
        Microsoft.Quantum.*
        Microsoft.Azure.Quantum.*
        ```
    
        References to packages have the following format:

        ```xml
        <PackageReference Include="Microsoft.Quantum.Compiler" Version="0.24.201332" />
        ```

    - Save the updated file.

    - Restore the dependencies of the project, by doing the following:

        ```dotnetcli
        dotnet restore [project_name].csproj
        ```

    > [!NOTE]
    > For versions `0.24.201332` and above, the target framework was upgraded from `netcoreapp3.1` to `net6.0` (except for libraries). If using an older QDK, you should keep this value as is.

5. Navigate back to the folder containing your main project and run:

    ```dotnetcli
    dotnet build [project_name].csproj
    ```

With your Q# projects now updated, follow the instructions below to update the QDK itself.

## Update the QDK

The process to update the QDK varies depending on your development language and environment.
Select your development environment below.

* [Python: update the `qsharp` package](#update-the-qsharp-python-package)
* [Jupyter Notebooks: update the IQ# kernel](#update-the-iq-jupyter-kernel)
* [Visual Studio: update the QDK extension](#update-the-qdk-for-visual-studio-extension)
* [VS Code: update the QDK extension](#update-the-qdk-for-vs-code-extension)
* [Command line and C#: update project templates](#c-using-the-dotnet-command-line-tool)
* [Python: Update the *azure-quantum* Python package](#update-the-azure-quantum-python-package)


## Update the `qsharp` Python package

The update procedure depends on whether you originally installed using conda or using the .NET CLI and pip.

### [Update using conda (recommended)](#tab/tabid-conda)

1. Activate the conda environment where you installed the `qsharp` package, and then run this command to update it:

    ```
    conda update -c microsoft qsharp
    ```

1. Run the following command from the location of your `.qs` files:

    ```
    python -c "import qsharp; qsharp.reload()"
    ```

### [Update using .NET CLI and pip (advanced)](#tab/tabid-dotnetcli)

1. Update the `iqsharp` kernel 

    ```dotnetcli
    dotnet tool update -g Microsoft.Quantum.IQSharp
    dotnet iqsharp install
    ```

1. Verify the `iqsharp` version

    ```dotnetcli
    dotnet iqsharp --version
    ```

    You should see the following output:

    ```
    iqsharp: 0.24.201332
    Jupyter Core: 1.5.0.0
    ```

    Don't worry if your `iqsharp` version is higher. It should match the [latest release](xref:microsoft.quantum.relnotes-qdk).

1. Update the `qsharp` package:

    ```
    pip install --upgrade qsharp
    ```

1. Verify the `qsharp` version:

    ```
    pip show qsharp
    ```

    You should see the following output:

    ```
    Name: qsharp
    Version: 0.24.201332
    Summary: Python client for Q#, a domain-specific quantum programming language
    ...
    ```

1. Run the following command from the location of your `.qs` files:

    ```
    python -c "import qsharp; qsharp.reload()"
    ```

***

You can now use the updated `qsharp` Python package to run your existing Python quantum programs.

## Update the IQ# Jupyter kernel

The update procedure depends on whether you originally installed using conda or using the .NET CLI and pip.

### [Update using conda (recommended)](#tab/tabid-conda)

1. Activate the conda environment where you installed the `qsharp` package, and then run this command to update it:

    ```
    conda update -c microsoft qsharp
    ```

1. Run the following command from a cell in each of your existing Q# Jupyter Notebooks:

    ```
    %workspace reload
    ```

### [Update using .NET CLI and pip (advanced)](#tab/tabid-dotnetcli)

1. Update the `Microsoft.Quantum.IQSharp` package:

    ```dotnetcli
    dotnet tool update -g Microsoft.Quantum.IQSharp
    dotnet iqsharp install
    ```

1. Verify the `iqsharp` version:

    ```dotnetcli
    dotnet iqsharp --version
    ```

    Your output should be similar to the following:

    ```
    iqsharp: 0.24.201332
    Jupyter Core: 1.5.0.0
    ```

    Don't worry if your `iqsharp` version is higher. It should match the [latest release](xref:microsoft.quantum.relnotes-qdk).

1. Run the following command from a cell in each of your existing Q# Jupyter Notebooks:

    ```
    %workspace reload
    ```

***

You can now use the updated IQ# kernel to run your existing Q# Jupyter Notebooks.

## Update the QDK for Visual Studio extension

1. Update the QDK for Visual Studio extension

    - Visual Studio prompts you to accept updates to the [QDK for Visual Studio extension](https://marketplace.visualstudio.com/items?itemName=quantum.DevKit64)
    - Accept the update

    > [!NOTE]
    > The project templates are updated with the extension. The updated templates apply to newly created projects only. The code for your existing projects is not updated when the extension is updated.

## Update the QDK for VS Code extension

1. Update the QDK for VS Code extension

    - Restart VS Code
    - Navigate to the **Extensions** tab
    - Select the **Microsoft Quantum Development Kit for Visual Studio Code** extension
    - Reload the extension

## C# using the `dotnet` command-line tool

1. Update the QDK project templates for .NET

    From the command prompt:

    ```dotnetcli
    dotnet new install Microsoft.Quantum.ProjectTemplates
    ```

   Alternatively, if you intend to use the command-line templates, and already have the QDK extension for VS Code installed, you can update the project templates from the extension itself:

   - [Update the QDK for VS Code extension](#update-the-qdk-for-vs-code-extension)
   - In VS Code, go to **View** -> **Command Palette**
   - Select **Q#: Install command line project templates**
   - After a few seconds you should get a popup confirming "project templates installed successfully"

## Update the azure-quantum Python package

1. Update to the latest `azure-quantum` Python package by using the package installer for Python (pip)

   ```Shell
   pip install --upgrade azure-quantum
   ```
   
1. If you encounter any issues please ensure that Python and pip are up to date. For information on the latest version requirements, [Install the azure-quantum Python package](xref:microsoft.quantum.install-qdk.overview)
