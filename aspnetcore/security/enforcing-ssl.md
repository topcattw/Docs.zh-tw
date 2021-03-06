---
title: "強制執行的 ASP.NET Core 應用程式中的 SSL"
author: rick-anderson
description: "示範如何要求使用 SSL 在 ASP.NET Core web 應用程式"
keywords: "ASP.NET Core、 SSL、 HTTPS、 RequireHttpsAttribute、 IIS Express"
ms.author: riande
manager: wpickett
ms.date: 07/19/2017
ms.topic: article
ms.technology: aspnet
ms.prod: asp.net-core
uid: security/enforcing-ssl
ms.openlocfilehash: 6f2755a606000717ca8a57f045b1ef613c7f14f6
ms.sourcegitcommit: 6e83c55eb0450a3073ef2b95fa5f5bcb20dbbf89
ms.translationtype: MT
ms.contentlocale: zh-TW
ms.lasthandoff: 09/28/2017
---
# <a name="enforcing-ssl-in-an-aspnet-core-app"></a>強制執行的 ASP.NET Core 應用程式中的 SSL

作者：[Rick Anderson](https://twitter.com/RickAndMSFT)

本文件說明如何：

- 需要 SSL （HTTPS 要求） 的所有要求。
- 將所有 HTTP 要求重新都導向至 HTTPS。

## <a name="require-ssl"></a>需要 SSL

[RequireHttpsAttribute](https://docs.microsoft.com/aspnet/core/api/microsoft.aspnetcore.mvc.requirehttpsattribute)用來要求 SSL。 可以用來裝飾控制器或具有此屬性的方法，或者您可以將它套用全域如下所示：

將下列程式碼加入`ConfigureServices`中`Startup`:

[!code-csharp[Main](authentication/accconfirm/sample/WebApp1/Startup.cs?name=snippet2&highlight=4-)]

上述反白顯示的程式碼需要的所有要求都使用`HTTPS`，因此 HTTP 要求都會被忽略。 下列反白顯示的程式碼會將所有 HTTP 要求重新都導向至 HTTPS:

[!code-csharp[Main](authentication/accconfirm/sample/WebApp1/Startup.cs?name=snippet_AddRedirectToHttps&highlight=7-)]

請參閱[URL 重寫中介軟體](xref:fundamentals/url-rewriting)如需詳細資訊。

全域需要 HTTPS (`options.Filters.Add(new RequireHttpsAttribute());`) 是安全性最佳作法。 套用`[RequireHttps]`所有控制器的屬性不會視為為全域需要 HTTPS 的安全。 您無法保證新的控制站新增至您的應用程式將會記住要套用`[RequireHttps]`屬性。
