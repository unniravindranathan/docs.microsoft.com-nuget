---
# required metadata

title: NuGet Credential Providers for Visual Studio | Microsoft Docs
author: kraigb
ms.author: kraigb
manager: ghogen
ms.date: 1/9/2017
ms.topic: article
ms.prod: nuget
#ms.service:
ms.technology: nuget
ms.assetid: 9c7f6d16-f437-47c4-82d4-6c996e0b18ec

# optional metadata

#description:
#keywords:
#ROBOTS:
#audience:
#ms.devlang:
ms.reviewer:
- karann
- harikm
#ms.suite:
#ms.tgt_pltfrm:
#ms.custom:

---

# NuGet credential providers for Visual Studio

The NuGet Visual Studio Extension 3.6+ supports credential providers, which enable NuGet to work with authenticated feeds.
After you install a NuGet credential provider for Visual Studio, the NuGet Visual Studio extension will automatically acquire and refresh credentials for authenticated feeds as necessary.

A sample implementation can be found in [the VsCredentialProvider sample](https://github.com/NuGet/Samples/tree/master/VsCredentialProvider).

> [!Note]
> NuGet credential providers for Visual Studio must be installed as a regular Visual Studio extension and will require [Visual Studio 2017](https://aka.ms/vs/15/preview/vs_enterprise) (currently in preview) or above.
>
> NuGet credential providers for Visual Studio work only in Visual Studio (not in dotnet restore or nuget.exe). For credential providers with nuget.exe, see [NuGet.exe Credential Providers](../api/nuget.exe-credential-providers.md).

## Available NuGet credential providers for Visual Studio

There is a credential provider built into the Visual Studio NuGet extension to support Visual Studio Team Services.

The NuGet Visual Studio Extension uses an internal `VsCredentialProviderImporter` which also scans for plug-in credential providers. These plug-in credential providers must be discoverable as a MEF Export of type `IVsCredentialProvider`.

Available plug-in credential providers include:

- [MyGet Credential Provider for Visual Studio 2017](http://docs.myget.org/docs/reference/credential-provider-for-visual-studio)

## Creating a NuGet credential provider for Visual Studio

The NuGet Visual Studio Extension 3.6+ implements an internal CredentialService that is used to acquire credentials. The CredentialService has a list of built-in and plug-in credential providers. Each provider is tried sequentially until credentials are acquired.

During credential acquisition, the credential service will try credential providers in the following order, stopping as soon as credentials are acquired:

1. Credentials will be fetched from NuGet configuration files (using the built-in `SettingsCredentialProvider`).
1. If the package source is on Visual Studio Team Services, the `VisualStudioAccountProvider` will be used.
1. All other plug-in credential providers will be tried sequentially.
1. If no credentials have been acquired yet, the user will be prompted for credentials using a standard basic authentication dialog.

### Implementing IVsCredentialProvider.GetCredentialsAsync

To create a NuGet credential provider for Visual Studio, create a Visual Studio Extension that exposes a public MEF Export implementing the `IVsCredentialProvider` type, and adheres to the principles outlined below.

    public interface IVsCredentialProvider
    {
        Task<ICredentials> GetCredentialsAsync(
          Uri uri,
          IWebProxy proxy,
          bool isProxyRequest,
          bool isRetry,
          bool nonInteractive,
          CancellationToken cancellationToken);
    }

A sample implementation can be found in [the VsCredentialProvider sample](https://github.com/NuGet/Samples/tree/master/VsCredentialProvider).

Each NuGet credential provider for Visual Studio must:

1. Determine whether it can provide credentials for the targeted URI before initiating credential acquisition. If the provider cannot supply credentials for the targeted source, then it should return `null`.
1. If the provider does handle requests for the targeted URI, but cannot supply credentials, an exception should be thrown.

A custom NuGet credential provider for Visual Studio must implement the `IVsCredentialProvider` interface available in the [NuGet.VisualStudio package](https://www.nuget.org/packages/NuGet.VisualStudio/).

#### GetCredentialAsync

| Input Parameter |Description|
| ----------------|-----------|
| Uri uri | The package source Uri for which credentials are being requested.|
| IWebProxy proxy | Web proxy to use when communicating on the network. Null if there is no proxy authentication configured. |
| bool isProxyRequest | True if this request is to get proxy authentication credentials. If the implementation is not valid for acquiring proxy credentials, then null should be returned. |
| bool isRetry | True if credentials were previously requested for this Uri, but the supplied credentials did not allow authorized access. |
| bool nonInteractive | If true, the credential provider must suppress all user prompts and use default values instead. |
| CancellationToken cancellationToken | This cancellation token should be checked to determine if the operation requesting credentials has been cancelled. |
  
**Return value**: A credentials object implementing the [`System.Net.ICredentials` interface](https://msdn.microsoft.com/en-us/library/system.net.icredentials.aspx).
