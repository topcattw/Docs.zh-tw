---
title: "ASP.NET Core WebListener web 伺服器實作"
author: rick-anderson
description: "導入了 WebListener，適用於在 Windows 上的 ASP.NET Core web 伺服器。 建置在 Http.Sys 核心模式驅動程式，WebListener 是可以用於 IIS 沒有網際網路直接連線的 Kestrel 的替代方式。"
keywords: "ASP.NET Core、 WebListener、 HttpListener、 url 前置詞，SSL"
ms.author: riande
manager: wpickett
ms.date: 08/07/2017
ms.topic: article
ms.assetid: 0a7286e4-6428-424e-b5c4-5c98815cf61c
ms.technology: aspnet
ms.prod: asp.net-core
uid: fundamentals/servers/weblistener
ms.openlocfilehash: bcd225875cfe2a544581c331231c704094780ea3
ms.sourcegitcommit: 74e22e08e3b08cb576e5184d16f4af5656c13c0c
ms.translationtype: MT
ms.contentlocale: zh-TW
ms.lasthandoff: 08/25/2017
---
# <a name="weblistener-web-server-implementation-in-aspnet-core"></a><span data-ttu-id="aff86-105">ASP.NET Core WebListener web 伺服器實作</span><span class="sxs-lookup"><span data-stu-id="aff86-105">WebListener web server implementation in ASP.NET Core</span></span>

<span data-ttu-id="aff86-106">由[Tom Dykstra](http://github.com/tdykstra)和[Chris Ross](https://github.com/Tratcher)</span><span class="sxs-lookup"><span data-stu-id="aff86-106">By [Tom Dykstra](http://github.com/tdykstra) and [Chris Ross](https://github.com/Tratcher)</span></span>

> [!NOTE]
> <span data-ttu-id="aff86-107">本主題只適用於 ASP.NET Core 1.x。</span><span class="sxs-lookup"><span data-stu-id="aff86-107">This topic applies only to ASP.NET Core 1.x.</span></span> <span data-ttu-id="aff86-108">在 ASP.NET Core 2.0 中，名為 WebListener [HTTP.sys](httpsys.md)。</span><span class="sxs-lookup"><span data-stu-id="aff86-108">In ASP.NET Core 2.0, WebListener is named [HTTP.sys](httpsys.md).</span></span>

<span data-ttu-id="aff86-109">WebListener 是[適用於 ASP.NET Core web 伺服器](index.md)只能在 Windows 上執行。</span><span class="sxs-lookup"><span data-stu-id="aff86-109">WebListener is a [web server for ASP.NET Core](index.md) that runs only on Windows.</span></span> <span data-ttu-id="aff86-110">此系統建置[Http.Sys 核心模式驅動程式](https://msdn.microsoft.com/library/windows/desktop/aa364510.aspx)。</span><span class="sxs-lookup"><span data-stu-id="aff86-110">It's built on the [Http.Sys kernel mode driver](https://msdn.microsoft.com/library/windows/desktop/aa364510.aspx).</span></span> <span data-ttu-id="aff86-111">WebListener 是替代[Kestrel](kestrel.md) ，可用來直接連線到網際網路不需依賴 IIS 做為反向 proxy 伺服器。</span><span class="sxs-lookup"><span data-stu-id="aff86-111">WebListener is an alternative to [Kestrel](kestrel.md) that can be used for direct connection to the Internet without relying on IIS as a reverse proxy server.</span></span> <span data-ttu-id="aff86-112">事實上， **WebListener 不能與 IIS 或 IIS Express，並不相容於[ASP.NET 核心模組](aspnet-core-module.md)。**</span><span class="sxs-lookup"><span data-stu-id="aff86-112">In fact, **WebListener can't be used with IIS or IIS Express, as it isn't compatible with the [ASP.NET Core Module](aspnet-core-module.md).**</span></span>

<span data-ttu-id="aff86-113">雖然 WebListener 針對 ASP.NET Core 開發，它可以用任何.NET Core 或.NET Framework 應用程式透過直接在[Microsoft.Net.Http.Server](https://www.nuget.org/packages/Microsoft.Net.Http.Server/) NuGet 封裝。</span><span class="sxs-lookup"><span data-stu-id="aff86-113">Although WebListener was developed for ASP.NET Core, it can be used directly in any .NET Core or .NET Framework application via the [Microsoft.Net.Http.Server](https://www.nuget.org/packages/Microsoft.Net.Http.Server/) NuGet package.</span></span>

<span data-ttu-id="aff86-114">WebListener 支援下列功能：</span><span class="sxs-lookup"><span data-stu-id="aff86-114">WebListener supports the following features:</span></span>

- [<span data-ttu-id="aff86-115">Windows 驗證</span><span class="sxs-lookup"><span data-stu-id="aff86-115">Windows Authentication</span></span>](xref:security/authentication/windowsauth)
- <span data-ttu-id="aff86-116">連接埠共用</span><span class="sxs-lookup"><span data-stu-id="aff86-116">Port sharing</span></span>
- <span data-ttu-id="aff86-117">SNI 使用 HTTPS</span><span class="sxs-lookup"><span data-stu-id="aff86-117">HTTPS with SNI</span></span>
- <span data-ttu-id="aff86-118">HTTP/2 5060，TLS (Windows 10)</span><span class="sxs-lookup"><span data-stu-id="aff86-118">HTTP/2 over TLS (Windows 10)</span></span>
- <span data-ttu-id="aff86-119">直接檔案傳輸</span><span class="sxs-lookup"><span data-stu-id="aff86-119">Direct file transmission</span></span>
- <span data-ttu-id="aff86-120">回應快取</span><span class="sxs-lookup"><span data-stu-id="aff86-120">Response caching</span></span>
- <span data-ttu-id="aff86-121">WebSockets (Windows 8)</span><span class="sxs-lookup"><span data-stu-id="aff86-121">WebSockets (Windows 8)</span></span>

<span data-ttu-id="aff86-122">支援的 Windows 版本：</span><span class="sxs-lookup"><span data-stu-id="aff86-122">Supported Windows versions:</span></span>

- <span data-ttu-id="aff86-123">Windows 7 和 Windows Server 2008 R2 和更新版本</span><span class="sxs-lookup"><span data-stu-id="aff86-123">Windows 7 and Windows Server 2008 R2 and later</span></span>

[<span data-ttu-id="aff86-124">檢視或下載範例程式碼</span><span class="sxs-lookup"><span data-stu-id="aff86-124">View or download sample code</span></span>](https://github.com/aspnet/Docs/tree/master/aspnetcore/fundamentals/servers/weblistener/sample)

## <a name="when-to-use-weblistener"></a><span data-ttu-id="aff86-125">何時使用 WebListener</span><span class="sxs-lookup"><span data-stu-id="aff86-125">When to use WebListener</span></span>

<span data-ttu-id="aff86-126">WebListener 可用於部署您要公開，直接向網際網路伺服器而不使用 IIS。</span><span class="sxs-lookup"><span data-stu-id="aff86-126">WebListener is useful for deployments where you need to expose the server directly to the Internet without using IIS.</span></span>

![Weblistener 直接與網際網路通訊](weblistener/_static/weblistener-to-internet.png)

<span data-ttu-id="aff86-128">由於它建置在 Http.Sys，WebListener 不需要保護以對抗攻擊反向 proxy 伺服器。</span><span class="sxs-lookup"><span data-stu-id="aff86-128">Because it's built on Http.Sys, WebListener doesn't require a reverse proxy server for protection against attacks.</span></span> <span data-ttu-id="aff86-129">Http.Sys 是成熟的技術，可保護免於許多種類的攻擊，並提供強固性、 安全性及功能完整的 web 伺服器的延展性。</span><span class="sxs-lookup"><span data-stu-id="aff86-129">Http.Sys is mature technology that protects against many kinds of attacks and provides the robustness, security, and scalability of a full-featured web server.</span></span> <span data-ttu-id="aff86-130">為 HTTP 接聽程式在 Http.Sys 之上，IIS 本身的執行。</span><span class="sxs-lookup"><span data-stu-id="aff86-130">IIS itself runs as an HTTP listener on top of Http.Sys.</span></span> 

<span data-ttu-id="aff86-131">WebListener 也是不錯的選擇內部部署，當您需要的功能，您無法使用 Kestrel 來取得它所提供的其中一個。</span><span class="sxs-lookup"><span data-stu-id="aff86-131">WebListener is also a good choice for internal deployments when you need one of the features it offers that you can't get by using Kestrel.</span></span>

![Weblistener 會直接與您的內部網路進行通訊](weblistener/_static/weblistener-to-internal.png)

## <a name="how-to-use-weblistener"></a><span data-ttu-id="aff86-133">如何使用 WebListener</span><span class="sxs-lookup"><span data-stu-id="aff86-133">How to use WebListener</span></span>

<span data-ttu-id="aff86-134">以下是針對主機作業系統和您的 ASP.NET Core 應用程式安裝工作的概觀。</span><span class="sxs-lookup"><span data-stu-id="aff86-134">Here's an overview of setup tasks for the host OS and your ASP.NET Core application.</span></span>

### <a name="configure-windows-server"></a><span data-ttu-id="aff86-135">設定 Windows Server</span><span class="sxs-lookup"><span data-stu-id="aff86-135">Configure Windows Server</span></span>

* <span data-ttu-id="aff86-136">安裝新版的.NET 應用程式所需，例如[.NET Core](https://go.microsoft.com/fwlink/?LinkID=827524)或.NET Framework 4.5.1。</span><span class="sxs-lookup"><span data-stu-id="aff86-136">Install the version of .NET that your application requires, such as [.NET Core](https://go.microsoft.com/fwlink/?LinkID=827524) or .NET Framework 4.5.1.</span></span>

* <span data-ttu-id="aff86-137">__'Asverify'__ 繫結至 WebListener，以及設定 SSL 憑證的 URL 前置詞</span><span class="sxs-lookup"><span data-stu-id="aff86-137">Preregister URL prefixes to bind to WebListener, and set up SSL certificates</span></span>

   <span data-ttu-id="aff86-138">如果您不要 __'asverify'__ URL 前置詞，在 Windows 中的，您必須以系統管理員權限執行您的應用程式。</span><span class="sxs-lookup"><span data-stu-id="aff86-138">If you don't preregister URL prefixes in Windows, you have to run your application with administrator privileges.</span></span> <span data-ttu-id="aff86-139">唯一的例外是如果您將繫結使用 HTTP (不是 HTTPS) 連接埠號碼大於 1024; localhost在此情況下，系統管理員權限不需要的。</span><span class="sxs-lookup"><span data-stu-id="aff86-139">The only exception is if you bind to localhost using HTTP (not HTTPS) with a port number greater than 1024; in that case administrator privileges aren't required.</span></span>

   <span data-ttu-id="aff86-140">如需詳細資訊，請參閱[如何 __'asverify'__ 前置詞及設定 SSL](#preregister-url-prefixes-and-configure-ssl)本文稍後。</span><span class="sxs-lookup"><span data-stu-id="aff86-140">For details, see [How to preregister prefixes and configure SSL](#preregister-url-prefixes-and-configure-ssl) later in this article.</span></span>

* <span data-ttu-id="aff86-141">開啟防火牆連接埠來允許流量到達 WebListener。</span><span class="sxs-lookup"><span data-stu-id="aff86-141">Open firewall ports to allow traffic to reach WebListener.</span></span>

   <span data-ttu-id="aff86-142">您可以使用 netsh.exe 或[PowerShell cmdlet](https://technet.microsoft.com/library/jj554906)。</span><span class="sxs-lookup"><span data-stu-id="aff86-142">You can use netsh.exe or [PowerShell cmdlets](https://technet.microsoft.com/library/jj554906).</span></span>

<span data-ttu-id="aff86-143">另外還有[Http.Sys 登錄設定](https://support.microsoft.com/kb/820129)。</span><span class="sxs-lookup"><span data-stu-id="aff86-143">There are also [Http.Sys registry settings](https://support.microsoft.com/kb/820129).</span></span>

### <a name="configure-your-aspnet-core-application"></a><span data-ttu-id="aff86-144">設定 ASP.NET Core 應用程式</span><span class="sxs-lookup"><span data-stu-id="aff86-144">Configure your ASP.NET Core application</span></span>

* <span data-ttu-id="aff86-145">安裝 NuGet 套件[Microsoft.AspNetCore.Server.WebListener](https://www.nuget.org/packages/Microsoft.AspNetCore.Server.WebListener/)。</span><span class="sxs-lookup"><span data-stu-id="aff86-145">Install the NuGet package [Microsoft.AspNetCore.Server.WebListener](https://www.nuget.org/packages/Microsoft.AspNetCore.Server.WebListener/).</span></span> <span data-ttu-id="aff86-146">這樣也會安裝[Microsoft.Net.Http.Server](https://www.nuget.org/packages/Microsoft.Net.Http.Server/)做為相依性。</span><span class="sxs-lookup"><span data-stu-id="aff86-146">This also installs [Microsoft.Net.Http.Server](https://www.nuget.org/packages/Microsoft.Net.Http.Server/) as a dependency.</span></span>

* <span data-ttu-id="aff86-147">呼叫[ `UseWebListener` ](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Hosting/WebHostBuilderKestrelExtensions/index.html#Microsoft.AspNetCore.Hosting.WebHostBuilderWebListenerExtensions.UseWebListener.md)上的擴充方法[WebHostBuilder](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Hosting/WebHostBuilder/index.html#Microsoft.AspNetCore.Hosting.WebHostBuilder.md)中您`Main`方法，並指定任何 WebListener[選項](https://github.com/aspnet/HttpSysServer/blob/rel/1.1.2/src/Microsoft.AspNetCore.Server.WebListener/WebListenerOptions.cs)和[設定](https://github.com/aspnet/HttpSysServer/blob/rel/1.1.2/src/Microsoft.Net.Http.Server/WebListenerSettings.cs)，您需要如下列範例所示：</span><span class="sxs-lookup"><span data-stu-id="aff86-147">Call the [`UseWebListener`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Hosting/WebHostBuilderKestrelExtensions/index.html#Microsoft.AspNetCore.Hosting.WebHostBuilderWebListenerExtensions.UseWebListener.md) extension method on [WebHostBuilder](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Hosting/WebHostBuilder/index.html#Microsoft.AspNetCore.Hosting.WebHostBuilder.md) in your `Main` method, specifying any WebListener [options](https://github.com/aspnet/HttpSysServer/blob/rel/1.1.2/src/Microsoft.AspNetCore.Server.WebListener/WebListenerOptions.cs) and [settings](https://github.com/aspnet/HttpSysServer/blob/rel/1.1.2/src/Microsoft.Net.Http.Server/WebListenerSettings.cs) that you need, as shown in the following example:</span></span>

  [!code-csharp[](weblistener/sample/Program.cs?name=snippet_Main&highlight=13-17)]

* <span data-ttu-id="aff86-148">設定 Url 和連接埠上接聽</span><span class="sxs-lookup"><span data-stu-id="aff86-148">Configure URLs and ports to listen on</span></span> 

  <span data-ttu-id="aff86-149">根據預設 ASP.NET Core 繫結至`http://localhost:5000`。</span><span class="sxs-lookup"><span data-stu-id="aff86-149">By default ASP.NET Core binds to `http://localhost:5000`.</span></span> <span data-ttu-id="aff86-150">若要設定 URL 前置詞和連接埠，您可以使用`UseURLs`擴充方法，`urls`命令列引數或 ASP.NET Core 組態系統。</span><span class="sxs-lookup"><span data-stu-id="aff86-150">To configure URL prefixes and ports, you can use the `UseURLs` extension method, the `urls` command-line argument or the ASP.NET Core configuration system.</span></span> <span data-ttu-id="aff86-151">如需詳細資訊，請參閱[主控](../../fundamentals/hosting.md)。</span><span class="sxs-lookup"><span data-stu-id="aff86-151">For more information, see [Hosting](../../fundamentals/hosting.md).</span></span>

  <span data-ttu-id="aff86-152">網頁接聽程式會使用[Http.Sys 前置詞字串格式](https://msdn.microsoft.com/library/windows/desktop/aa364698.aspx)。</span><span class="sxs-lookup"><span data-stu-id="aff86-152">Web Listener uses the [Http.Sys prefix string formats](https://msdn.microsoft.com/library/windows/desktop/aa364698.aspx).</span></span> <span data-ttu-id="aff86-153">沒有前置詞字串格式需求 WebListener 特有的。</span><span class="sxs-lookup"><span data-stu-id="aff86-153">There are no prefix string format requirements that are specific to WebListener.</span></span>

  > [!NOTE]
  > <span data-ttu-id="aff86-154">請確定您指定在相同的前置詞字串`UseUrls`，您預先註冊的伺服器上。</span><span class="sxs-lookup"><span data-stu-id="aff86-154">Make sure that you specify the same prefix strings in `UseUrls` that you preregister on the server.</span></span> 

* <span data-ttu-id="aff86-155">請確定您的應用程式未設定為執行 IIS 或 IIS Express。</span><span class="sxs-lookup"><span data-stu-id="aff86-155">Make sure your application is not configured to run IIS or IIS Express.</span></span>

  <span data-ttu-id="aff86-156">在 Visual Studio 中的預設啟動設定檔為 IIS Express。</span><span class="sxs-lookup"><span data-stu-id="aff86-156">In Visual Studio, the default launch profile is for IIS Express.</span></span>  <span data-ttu-id="aff86-157">若要執行為主控台應用程式的專案您必須手動變更選取的設定檔，如下列螢幕擷取畫面所示。</span><span class="sxs-lookup"><span data-stu-id="aff86-157">To run the project as a console application you have to manually change the selected profile, as shown in the following screen shot.</span></span>

  ![選取主控台應用程式設定檔](weblistener/_static/vs-choose-profile.png)

## <a name="how-to-use-weblistener-outside-of-aspnet-core"></a><span data-ttu-id="aff86-159">如何使用 ASP.NET Core 之外 WebListener</span><span class="sxs-lookup"><span data-stu-id="aff86-159">How to use WebListener outside of ASP.NET Core</span></span>

* <span data-ttu-id="aff86-160">安裝[Microsoft.Net.Http.Server](https://www.nuget.org/packages/Microsoft.Net.Http.Server/) NuGet 封裝。</span><span class="sxs-lookup"><span data-stu-id="aff86-160">Install the [Microsoft.Net.Http.Server](https://www.nuget.org/packages/Microsoft.Net.Http.Server/) NuGet package.</span></span>

* <span data-ttu-id="aff86-161">[__'Asverify'__ 繫結至 WebListener，以及設定 SSL 憑證的 URL 前置詞](#preregister-url-prefixes-and-configure-ssl)以用於 ASP.NET Core 一樣。</span><span class="sxs-lookup"><span data-stu-id="aff86-161">[Preregister URL prefixes to bind to WebListener, and set up SSL certificates](#preregister-url-prefixes-and-configure-ssl) as you would for use in ASP.NET Core.</span></span>

<span data-ttu-id="aff86-162">另外還有[Http.Sys 登錄設定](https://support.microsoft.com/kb/820129)。</span><span class="sxs-lookup"><span data-stu-id="aff86-162">There are also [Http.Sys registry settings](https://support.microsoft.com/kb/820129).</span></span>


<span data-ttu-id="aff86-163">以下是示範 WebListener 使用 ASP.NET Core 之外的程式碼範例：</span><span class="sxs-lookup"><span data-stu-id="aff86-163">Here's a code sample that demonstrates WebListener use outside of ASP.NET Core:</span></span>

```csharp
var settings = new WebListenerSettings();
settings.UrlPrefixes.Add("http://localhost:8080");

using (WebListener listener = new WebListener(settings))
{
    listener.Start();

    while (true)
    {
        var context = await listener.AcceptAsync();
        byte[] bytes = Encoding.ASCII.GetBytes("Hello World: " + DateTime.Now);
        context.Response.ContentLength = bytes.Length;
        context.Response.ContentType = "text/plain";

        await context.Response.Body.WriteAsync(bytes, 0, bytes.Length);
        context.Dispose();
    }
}
```

## <a name="preregister-url-prefixes-and-configure-ssl"></a><span data-ttu-id="aff86-164">__'Asverify'__ URL 前置詞及設定 SSL</span><span class="sxs-lookup"><span data-stu-id="aff86-164">Preregister URL prefixes and configure SSL</span></span>

<span data-ttu-id="aff86-165">IIS 和 WebListener 依賴接聽要求的基礎 Http.Sys 核心模式驅動程式，並執行初始處理。</span><span class="sxs-lookup"><span data-stu-id="aff86-165">Both IIS and WebListener rely on the underlying Http.Sys kernel mode driver to listen for requests and do initial processing.</span></span> <span data-ttu-id="aff86-166">在 IIS 中，管理 UI 可讓您很輕鬆地設定所有項目。</span><span class="sxs-lookup"><span data-stu-id="aff86-166">In IIS, the management UI gives you a relatively easy way to configure everything.</span></span> <span data-ttu-id="aff86-167">不過，如果您使用 WebListener 您需要自行設定 Http.Sys。</span><span class="sxs-lookup"><span data-stu-id="aff86-167">However, if you're using WebListener you need to configure Http.Sys yourself.</span></span> <span data-ttu-id="aff86-168">也就是執行 netsh.exe 內建的工具。</span><span class="sxs-lookup"><span data-stu-id="aff86-168">The built-in tool for doing that is netsh.exe.</span></span> 

<span data-ttu-id="aff86-169">您必須使用 netsh.exe 的最常見的工作會保留 URL 首碼，並指派 SSL 憑證。</span><span class="sxs-lookup"><span data-stu-id="aff86-169">The most common tasks you need to use netsh.exe for are reserving URL prefixes and assigning SSL certificates.</span></span>

<span data-ttu-id="aff86-170">NetSh.exe 不是使用適用於初學者簡單工具。</span><span class="sxs-lookup"><span data-stu-id="aff86-170">NetSh.exe is not an easy tool to use for beginners.</span></span> <span data-ttu-id="aff86-171">下列範例會示範保留連接埠 80 和 443 的 URL 前置詞時所需的最低限度：</span><span class="sxs-lookup"><span data-stu-id="aff86-171">The following example shows the bare minimum needed to reserve URL prefixes for ports 80 and 443:</span></span>

```console
netsh http add urlacl url=http://+:80/ user=Users
netsh http add urlacl url=https://+:443/ user=Users
```

<span data-ttu-id="aff86-172">下列範例會示範如何將 SSL 憑證指派：</span><span class="sxs-lookup"><span data-stu-id="aff86-172">The following example shows how to assign an SSL certificate:</span></span>

```console
netsh http add sslcert ipport=0.0.0.0:443 certhash=MyCertHash_Here appid={00000000-0000-0000-0000-000000000000}".
```

<span data-ttu-id="aff86-173">以下是正式的參考文件：</span><span class="sxs-lookup"><span data-stu-id="aff86-173">Here is the official reference documentation:</span></span>

* [<span data-ttu-id="aff86-174">Netsh 命令都會使用超文字傳輸通訊協定 (HTTP)</span><span class="sxs-lookup"><span data-stu-id="aff86-174">Netsh Commands for Hypertext Transfer Protocol (HTTP)</span></span>](http://technet.microsoft.com/library/cc725882.aspx)
* [<span data-ttu-id="aff86-175">UrlPrefix 字串</span><span class="sxs-lookup"><span data-stu-id="aff86-175">UrlPrefix Strings</span></span>](https://msdn.microsoft.com/library/windows/desktop/aa364698.aspx)

<span data-ttu-id="aff86-176">下列資源提供數種案例的詳細的指示。</span><span class="sxs-lookup"><span data-stu-id="aff86-176">The following resources provide detailed instructions for several scenarios.</span></span> <span data-ttu-id="aff86-177">參考的文件`HttpListener`同樣適用於`WebListener`，因為兩者都以 Http.Sys。</span><span class="sxs-lookup"><span data-stu-id="aff86-177">Articles that refer to `HttpListener` apply equally to `WebListener`, as both are based on Http.Sys.</span></span>

* [<span data-ttu-id="aff86-178">如何： 使用 SSL 憑證設定連接埠</span><span class="sxs-lookup"><span data-stu-id="aff86-178">How to: Configure a Port with an SSL Certificate</span></span>](http://msdn.microsoft.com/library/ms733791.aspx)
* <span data-ttu-id="aff86-179">[HTTPS 通訊-HttpListener 基礎代管與用戶端憑證](http://sunshaking.blogspot.com/2012/11/https-communication-httplistener-based.html)這是協力廠商架構部落格和相當老舊，但仍有有用的資訊。</span><span class="sxs-lookup"><span data-stu-id="aff86-179">[HTTPS Communication - HttpListener based Hosting and Client Certification](http://sunshaking.blogspot.com/2012/11/https-communication-httplistener-based.html) This is a third-party blog and is fairly old but still has useful information.</span></span>
* <span data-ttu-id="aff86-180">[如何： 逐步解說使用 HttpListener 或 Http 伺服器 unmanaged 程式碼 （c + +） 做為 SSL 簡單伺服器](http://blogs.msdn.com/b/jpsanders/archive/2009/09/29/walkthrough-using-httplistener-as-an-ssl-simple-server.aspx)這也是較舊的部落格包含有用資訊。</span><span class="sxs-lookup"><span data-stu-id="aff86-180">[How To: Walkthrough Using HttpListener or Http Server unmanaged code (C++) as an SSL Simple Server](http://blogs.msdn.com/b/jpsanders/archive/2009/09/29/walkthrough-using-httplistener-as-an-ssl-simple-server.aspx) This too is an older blog with useful information.</span></span>
* [<span data-ttu-id="aff86-181">如何設定 SSL 使用.NET 核心 WebListener？</span><span class="sxs-lookup"><span data-stu-id="aff86-181">How Do I Set Up A .NET Core WebListener With SSL?</span></span>](https://blogs.msdn.microsoft.com/timomta/2016/11/04/how-do-i-set-up-a-net-core-weblistener-with-ssl/)

<span data-ttu-id="aff86-182">以下是一些可以比使用 netsh.exe 命令列的協力廠商工具。</span><span class="sxs-lookup"><span data-stu-id="aff86-182">Here are some third-party tools that can be easier to use than the netsh.exe command line.</span></span> <span data-ttu-id="aff86-183">這些不是所提供，或經由 Microsoft 背書。</span><span class="sxs-lookup"><span data-stu-id="aff86-183">These are not provided by or endorsed by Microsoft.</span></span> <span data-ttu-id="aff86-184">這些工具執行系統管理員身分根據預設，由於 netsh.exe 本身需要系統管理員權限。</span><span class="sxs-lookup"><span data-stu-id="aff86-184">The tools run as administrator by default, since netsh.exe itself requires administrator privileges.</span></span>

* <span data-ttu-id="aff86-185">[http.sys 管理員](http://httpsysmanager.codeplex.com/)清單提供 UI，並設定 SSL 憑證和選項保留的前置詞及憑證信任清單。</span><span class="sxs-lookup"><span data-stu-id="aff86-185">[http.sys Manager](http://httpsysmanager.codeplex.com/) provides UI for listing and configuring SSL certificates and options, prefix reservations, and certificate trust lists.</span></span> 
* <span data-ttu-id="aff86-186">[HttpConfig](http://www.stevestechspot.com/ABetterHttpcfg.aspx)可讓您列出或設定 SSL 憑證和 URL 前置詞。</span><span class="sxs-lookup"><span data-stu-id="aff86-186">[HttpConfig](http://www.stevestechspot.com/ABetterHttpcfg.aspx) lets you list or configure SSL certificates and URL prefixes.</span></span> <span data-ttu-id="aff86-187">UI 會更精簡比 http.sys 管理員，並公開更多組態選項，但否則它會提供類似的功能。</span><span class="sxs-lookup"><span data-stu-id="aff86-187">The UI is more refined than http.sys Manager and exposes a few more configuration options, but otherwise it provides similar functionality.</span></span> <span data-ttu-id="aff86-188">它無法建立新的憑證信任清單 (CTL)，但可以將現有的指派。</span><span class="sxs-lookup"><span data-stu-id="aff86-188">It cannot create a new certificate trust list (CTL), but can assign existing ones.</span></span>

<span data-ttu-id="aff86-189">產生自我簽署的 SSL 憑證，Microsoft 提供的命令列工具： [MakeCert.exe](https://msdn.microsoft.com/library/windows/desktop/aa386968)和 PowerShell 指令程式[New-selfsignedcertificate](https://technet.microsoft.com/library/hh848633)。</span><span class="sxs-lookup"><span data-stu-id="aff86-189">For generating self-signed SSL certificates, Microsoft provides command-line tools: [MakeCert.exe](https://msdn.microsoft.com/library/windows/desktop/aa386968) and the PowerShell cmdlet [New-SelfSignedCertificate](https://technet.microsoft.com/library/hh848633).</span></span> <span data-ttu-id="aff86-190">另外還有第三方 UI 工具，可讓您更輕鬆地產生自我簽署的 SSL 憑證：</span><span class="sxs-lookup"><span data-stu-id="aff86-190">There are also third-party UI tools that make it easier for you to generate self-signed SSL certificates:</span></span>

* [<span data-ttu-id="aff86-191">SelfCert</span><span class="sxs-lookup"><span data-stu-id="aff86-191">SelfCert</span></span>](https://www.pluralsight.com/blog/software-development/selfcert-create-a-self-signed-certificate-interactively-gui-or-programmatically-in-net)
* [<span data-ttu-id="aff86-192">Makecert UI</span><span class="sxs-lookup"><span data-stu-id="aff86-192">Makecert UI</span></span>](http://makecertui.codeplex.com/)

## <a name="next-steps"></a><span data-ttu-id="aff86-193">後續步驟</span><span class="sxs-lookup"><span data-stu-id="aff86-193">Next steps</span></span>

<span data-ttu-id="aff86-194">如需詳細資訊，請參閱下列資源：</span><span class="sxs-lookup"><span data-stu-id="aff86-194">For more information, see the following resources:</span></span>

* [<span data-ttu-id="aff86-195">這篇文章的範例應用程式</span><span class="sxs-lookup"><span data-stu-id="aff86-195">Sample app for this article</span></span>](https://github.com/aspnet/Docs/tree/master/aspnetcore/fundamentals/servers/weblistener/sample)
* [<span data-ttu-id="aff86-196">WebListener 原始程式碼</span><span class="sxs-lookup"><span data-stu-id="aff86-196">WebListener source code</span></span>](https://github.com/aspnet/HttpSysServer/)
* [<span data-ttu-id="aff86-197">裝載</span><span class="sxs-lookup"><span data-stu-id="aff86-197">Hosting</span></span>](../hosting.md)