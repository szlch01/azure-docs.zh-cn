---
title: "使用 Azure 媒体服务来保护内容 | Microsoft 文档"
description: "本文概述了如何使用媒体服务来保护内容。"
services: media-services
documentationcenter: 
author: Juliako
manager: cfowler
editor: 
ms.assetid: 81bc00e1-dcda-4d69-b9ab-8768b793422b
ms.service: media-services
ms.workload: media
ms.tgt_pltfrm: na
ms.devlang: na
ms.topic: article
ms.date: 06/29/2017
ms.author: juliako
ms.openlocfilehash: 6475a865b9d1b263bd7cc68c99acdb5f6959531e
ms.sourcegitcommit: cc03e42cffdec775515f489fa8e02edd35fd83dc
ms.translationtype: HT
ms.contentlocale: zh-CN
ms.lasthandoff: 12/07/2017
---
# <a name="protecting-content-overview"></a>保护内容概述
使用 Microsoft Azure 媒体服务，可以在媒体从离开计算机到存储、处理和传送的整个过程中确保其安全。 借助媒体服务，可以传送使用高级加密标准 (AES-128) 或三个主要 DRM 系统（Microsoft PlayReady、Google Widevine 和 Apple FairPlay）中任意一个动态加密的实时和请求内容。 媒体服务还提供了用于向已授权客户端传送 AES 密钥和 DRM（PlayReady、Widevine 和 FairPlay）许可证的服务。 

下图阐释 Azure 媒体服务内容保护工作流。 

![使用 PlayReady 进行保护](./media/media-services-content-protection-overview/media-services-content-protection-with-multi-drm.png)

本文介绍与了解使用 AMS 进行内容保护相关的概念和术语。 本文还提供指向讨论如何保护内容的文章的链接。 

## <a name="dynamic-encryption"></a>动态加密
借助 Azure 媒体服务，可以传送使用 AES 明文密钥或 DRM 加密（Microsoft PlayReady、Google Widevine 和 Apple FairPlay）动态加密的内容。 当前可以加密以下流格式：HLS、MPEG DASH 和平滑流。 不支持对渐进式下载加密。 每个加密方法均支持以下流式处理协议：
- AES：MPEG-DASH、平滑流式处理和 HLS
- PlayReady：MPEG-DASH、平滑流式处理和 HLS
- Widevine：MPEG-DASH
- FairPlay：HLS

若要加密资产，则需要关联加密内容密钥和资产并且为该密钥配置授权策略。 可以指定或由媒体服务自动生成内容密钥。

还需要配置资产的传送策略。 如果要流式传输存储加密的资产，请确保通过配置资产传送策略来指定该资产的传送方式。

播放器请求流时，媒体服务将使用指定的密钥通过 AES 明文密钥或 DRM 加密来动态加密内容。 为了解密流，播放器从 AMS 密钥传送服务请求密钥。 为了确定用户是否被授权获取密钥，服务将评估你为密钥指定的授权策略。

## <a name="aes-128-clear-key-vs-drm"></a>AES-128 明文密钥与 DRM
客户通常希望知道他们应该使用 AES 加密还是 DRM 系统。 使用 AES 加密和 DRM 系统的主要区别是，使用 AES 加密时，内容密钥以未加密格式（“明文”）传输到客户端。 因此，可以通过网络跟踪在客户端上明文查看用于加密内容的密钥。 AES-128 明文密钥适合查看者是受信任方的用例（例如：员工查看公司内分发的加密公司视频）。

与 AES-128 明文密钥加密相比，PlayReady、Widevine 和 FairPlay 均提供更高等级的加密。 内容密钥以加密格式传输。 此外，解密是安全环境中操作系统级别的句柄，它使得恶意用户的攻击变得格外困难。 在查看者可能并非受信任方且需要更高等级的安全性的用例中，建议使用 DRM。

## <a name="storage-encryption"></a>存储加密
可以使用存储加密通过 AES-256 位加密在本地加密明文内容，然后将其上传到 Azure 存储以加密形式静态存储相关内容。 受存储加密保护的资产会在编码前自动解密并放入经过加密的文件系统中，并可选择在重新上传为新的输出资产前重新加密。 存储加密的主要用例是在磁盘上通过静态增强加密来保护高品质的输入媒体文件。

要传送存储加密资产，必须配置资产的传送策略，以使媒体服务了解要如何传送内容。 在流式传输资产之前，流式处理服务器会解密，然后再使用指定的传送策略（例如 AES、通用加密或无加密）流式传输内容。

## <a name="types-of-encryption"></a>加密类型
Playready 和 Widevine 使用通用加密（AES CTR 模式）。 FairPlay 使用 AES CBC 模式加密。 AES-128 明文密钥加密使用信封加密。

## <a name="licenses-and-keys-delivery-service"></a>许可证和密钥传送服务
媒体服务提供用于向已授权客户端传送 DRM（PlayReady、Widevine 和 FairPlay）许可证和 AES 密钥的密钥传送服务。 可以使用 [Azure 门户](media-services-portal-protect-content.md)、REST API 或适用于 .NET 的媒体服务 SDK 来配置许可证和密钥的授权与身份验证策略。

## <a name="control-content-access"></a>控制内容访问
可以通过配置内容密钥授权策略控制谁有权访问内容。 内容密钥授权策略支持开放或令牌限制。

### <a name="open-authorization"></a>开放授权
通过开放授权策略，将内容密钥发送到任意客户端（无限制）。

### <a name="token-authorization"></a>令牌授权
通过令牌限制授权策略，内容密钥将仅发送到在密钥/许可证请求中表示有效 JSON Web 令牌 (JWT) 或简单 Web 令牌 (SWT) 的客户端。 此令牌必须由安全令牌服务 (STS) 颁发。 可以将 Microsoft Active Directory 用作 STS 或者部署一个自定义 STS。 必须将 STS 配置为创建令牌，该令牌使用指定密钥以及在令牌限制配置中指定的颁发声明进行签名。 如果令牌有效，而且令牌中的声明与为密钥/许可证配置的声明相匹配，则媒体服务密钥传送服务会将请求的密钥/许可证返回到客户端。

在配置令牌限制策略时，必须指定主验证密钥、颁发者和受众参数。 主验证密钥包含用来为令牌签名的密钥，颁发者是颁发令牌的安全令牌服务。 受众（有时称为范围）描述该令牌的意图，或者令牌授权访问的资源。 媒体服务密钥交付服务会验证令牌中的这些值是否与模板中的值匹配。

## <a name="streaming-urls"></a>流 URL
如果使用了多个 DRM 加密资产，则应在流式处理 URL 中使用加密标记：(format='m3u8-aapl', encryption='xxx')。

请注意以下事项：
* 仅可以指定不多于一个加密类型。
* 如果资产仅应用了一种加密，则无需在该 URL 中指定加密类型。
* 加密类型不区分大小写。
* 可以指定以下加密类型：  
  * **cenc**：适用于 PlayReady 或 Widevine（通用加密）
  * **cbcs-aapl**：适用于 FairPlay（AES CBC 加密）
  * **cbc**：适用于 AES 信封加密（信封加密）

## <a name="next-steps"></a>后续步骤
下文介绍内容保护入门的后续步骤：
* [使用存储加密进行保护](media-services-rest-storage-encryption.md)
* [使用 AES 加密进行保护](media-services-protect-with-aes128.md)
* [使用 PlayReady 和/或 Widevine 进行保护](media-services-protect-with-playready-widevine.md)
* [使用 FairPlay 进行保护](media-services-protect-hls-with-FairPlay.md)

## <a name="related-links"></a>相关链接
[Azure 媒体服务 PlayReady 许可证交付定价详述](http://mingfeiy.com/playready-pricing-explained-in-azure-media-services)

[如何在 Azure 媒体服务中调用 AES 加密流](http://mingfeiy.com/debug-aes-encrypted-stream-azure-media-services)

[JWT 令牌身份验证](http://www.gtrifonov.com/2015/01/03/jwt-token-authentication-in-azure-media-services-and-dynamic-encryption/)

[将基于 Azure 媒体服务 OWIN MVC 的应用与 Azure Active Directory 相集成，并基于 JWT 声明限制内容密钥传送](http://www.gtrifonov.com/2015/01/24/mvc-owin-azure-media-services-ad-integration/)。

[content-protection]: ./media/media-services-content-protection-overview/media-services-content-protection.png
