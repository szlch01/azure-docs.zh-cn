---
title: "使用 Azure 资源管理器模板创建多 VM 环境和 PaaS 资源 | Microsoft Docs"
description: "了解如何在 Azure 开发测试实验室中通过 Azure 资源管理器模板创建多 VM 环境和 PaaS 资源"
services: devtest-lab,virtual-machines,visual-studio-online
documentationcenter: na
author: tomarcher
manager: douge
editor: 
ms.assetid: 
ms.service: devtest-lab
ms.workload: na
ms.tgt_pltfrm: na
ms.devlang: na
ms.topic: article
ms.date: 01/31/2017
ms.author: tarcher
ms.openlocfilehash: 381b3e5cc82e71dde4fe3a4266678dbd5919290d
ms.sourcegitcommit: b07d06ea51a20e32fdc61980667e801cb5db7333
ms.translationtype: HT
ms.contentlocale: zh-CN
ms.lasthandoff: 12/08/2017
---
# <a name="create-multi-vm-environments-and-paas-resources-with-azure-resource-manager-templates"></a>使用 Azure 资源管理器模板创建多 VM 环境和 PaaS 资源

使用 [Azure 门户](http://go.microsoft.com/fwlink/p/?LinkID=525040)可以轻松地[在实验室中创建和添加 VM](https://docs.microsoft.com/azure/devtest-lab/devtest-lab-add-vm)。 这种方法非常适合用于一次创建一个 VM。 但是，如果环境包含多个 VM，则必须单独创建每个 VM。 对于多层 Web 应用或 SharePoint 场等情况，需要使用某种机制以单个步骤创建多个 VM。 现在，可以使用 Azure 资源管理器模板定义 Azure 解决方案的基础结构和配置，以一致的状态重复部署多个 VM。 此功能提供以下优势：

- Azure 资源管理器模板是从源代码管理存储库（GitHub 或 Team Services Git）直接加载的。
- 配置后，用户可以在 Azure 门户中通过选择 Azure 资源管理器模板来创建环境，就像创建其他类型的 [VM 库](./devtest-lab-comparing-vm-base-image-types.md)一样。
- 除了 IaaS VM 以外，还可以通过 Azure 资源管理器模板在环境中预配 Azure PaaS 资源。
- 除了其他类型的库创建的单个 VM 以外，还可以在实验室中跟踪环境的成本。
- PaaS 资源会被创建，并在成本跟踪中显示；但 VM 自动关闭不适用于 PaaS 资源。
- 对于环境，用户可以实施他们针对单实验室 VM 所实施的相同的 VM 策略控制。

详细了解[使用 Resource Manager 模板](https://docs.microsoft.com/azure/azure-resource-manager/resource-group-overview#the-benefits-of-using-resource-manager)通过单个操作部署、更新或删除所有实验室资源的众多优点。

> [!NOTE]
> 基于 Resource Manager 模板创建更多实验室 VM 时，无论正创建多 VM 还是单 VM，都需注意某些差异。 “使用虚拟机的 Azure 资源管理器模板”更加详细地阐述这些差异。
>
>

## <a name="configure-azure-resource-manager-template-repositories"></a>配置 Azure 资源管理器模板存储库

在基础结构即代码和配置即代码方面，最佳做法之一是在源代码管理中管理环境模板。 Azure 开发测试实验室遵循这种做法，它直接从 GitHub 或 VSTS Git 存储库加载所有 Azure 资源管理器模板。 因此，从测试环境到生产环境，Resource Manager 模板可用于整个发布周期。

在存储库中整理 Azure 资源管理器模板时需遵循几条规则：

- 必须将主模板文件命名为 `azuredeploy.json`。 

    ![关键的 Azure 资源管理器模板文件](./media/devtest-lab-create-environment-from-arm/master-template.png)

- 如果想要使用参数文件中定义的参数值，必须将参数文件命名为 `azuredeploy.parameters.json`。
- 可以使用参数 `_artifactsLocation` 和 `_artifactsLocationSasToken` 来构造 parametersLink URI 的值，以允许开发测试实验室自动管理嵌套的模板。 请参阅 [How Azure DevTest Labs makes nested Resource Manager template deployments easier for testing environments](https://blogs.msdn.microsoft.com/devtestlab/2017/05/23/how-azure-devtest-labs-makes-nested-arm-template-deployments-easier-for-testing-environments/)（Azure 开发测试实验室如何使在测试环境中部署嵌套的资源管理器模板部署更轻松），了解详细信息。
- 可以定义元数据来指定模板显示名称和说明。 此元数据必须在名为 `metadata.json` 的文件中。 以下示例元数据文件演示如何指定显示名称和说明： 

```json
{
 
"itemDisplayName": "<your template name>",
 
"description": "<description of the template>"
 
}
```

以下步骤指导完成使用 Azure 门户将存储库添加到实验室的整个过程。 

1. 登录到 [Azure 门户](http://go.microsoft.com/fwlink/p/?LinkID=525040)。
1. 选择“更多服务”，并从列表中选择“开发测试实验室”。
1. 从实验室列表，选择所需的实验室。   
1. 在实验室的边栏选项卡中，选择“配置和策略”。

    ![配置和策略](./media/devtest-lab-create-environment-from-arm/configuration-and-policies-menu.png)

1. 从“配置和策略”设置列表中选择“存储库”。 “存储库”边栏选项卡列出已添加到实验室的存储库。 名为 `Public Repo` 的存储库是系统自动为所有实验室生成的，它连接到包含你所用的多个 VM 项目的[开发测试实验室 GitHub 存储库](https://github.com/Azure/azure-devtestlab)。

    ![公共存储库](./media/devtest-lab-create-environment-from-arm/public-repo.png)

1. 选择“添加+”来添加 Azure 资源管理器模板存储库。
1. 当第二个“存储库”边栏选项卡打开时，请按如下所示输入所需的信息：
    - **名称** - 输入实验室中使用的存储库名称。
    - **Git 克隆 URI** - 输入 GitHub 或 Visual Studio Team Services 中的 GIT HTTPS 克隆 URL。  
    - **分支** - 输入用于访问 Azure 资源管理器模板定义的分支名称。 
    - **个人访问令牌** - 个人访问令牌用于安全访问存储库。 要从 Visual Studio Team Services 获取令牌，请选择“&lt;姓名>”>“我的配置文件”>“安全”>“公共访问令牌”。 要从 GitHub 获取令牌，请选择头像，然后选择“设置”>“公共访问令牌”。 
    - **文件夹路径** - 使用两个输入字段中的一个，输入以正斜杠“/”开头的文件夹路径，该路径是项目定义（第一个输入字段）或 Azure 资源管理器模板定义的 Git 克隆 URI 的相对路径。   
    
        ![公共存储库](./media/devtest-lab-create-environment-from-arm/repo-values.png)

1. 输入所有必填字段并通过验证后，请选择“保存”。

下一部分将逐步讲解如何通过 Azure 资源管理器模板创建环境。

## <a name="create-an-environment-from-an-azure-resource-manager-template"></a>通过 Azure 资源管理器模板创建环境

在实验室中配置 Azure 资源管理器模板存储库后，实验室用户可在 Azure 门户中使用以下步骤创建环境：

1. 登录到 [Azure 门户](http://go.microsoft.com/fwlink/p/?LinkID=525040)。
1. 选择“更多服务”，并从列表中选择“开发测试实验室”。
1. 从实验室列表，选择所需的实验室。   
1. 在实验室的边栏选项卡中，选择“添加+”。
1. “选择库”边栏选项卡显示了可与最前面列出的 Azure 资源管理器模板配合使用的基本映像。 选择所需的 Azure 资源管理器模板。

    ![选择一个库](./media/devtest-lab-create-environment-from-arm/choose-a-base.png)
  
1. 在“添加”边栏选项卡中，输入“环境名称”值。 环境名称是向实验室中的用户显示的名称。 剩余的输入字段已在 Azure 资源管理器模板中定义。 如果模板中定义了默认值或者存在 `azuredeploy.parameter.json` 文件，默认值会显示在这些输入字段中。 对于*安全字符串*类型的参数，可以使用实验室的[个人机密存储](https://azure.microsoft.com/en-us/updates/azure-devtest-labs-keep-your-secrets-safe-and-easy-to-use-with-the-new-personal-secret-store)中存储的机密。

    ![“添加”边栏选项卡](./media/devtest-lab-create-environment-from-arm/add.png)

    > [!NOTE]
    > 有几个参数值会显示为空值，即使已指定这些值。 因此，如果用户将这些值分配到 Azure 资源管理器模板中的参数，开发测试实验室不会显示这些值，而是显示空白的输入字段，实验室用户在创建环境时需要输入相应的值。
    > 
    > - GEN-UNIQUE
    > - GEN-UNIQUE-[N]
    > - GEN-SSH-PUB-KEY
    > - GEN-PASSWORD 
 
1. 选择“添加”创建环境。 环境随即会开始预配，其状态显示在“我的虚拟机”列表中。 实验室会自动创建一个新资源组，用于预配 Azure 资源管理器模板中定义的所有资源。
1. 创建环境后，请在“我的虚拟机”列表中选择该环境，打开资源组边栏选项卡并浏览该环境中预配的所有资源。
    
    ![“我的虚拟机”列表](./media/devtest-lab-create-environment-from-arm/all-environment-resources.png)
   
   还可以展开环境仅查看该环境中预配的 VM 列表。
    
    ![“我的虚拟机”列表](./media/devtest-lab-create-environment-from-arm/my-vm-list.png)

1. 单击任一环境可以查看可用的操作 - 例如，应用项目、附加数据磁盘、更改自动关闭时间，等等。

    ![环境操作](./media/devtest-lab-create-environment-from-arm/environment-actions.png)

## <a name="next-steps"></a>后续步骤
* 创建 VM 后，可通过选择 VM 边栏选项卡中的“连接”来连接该 VM。
* 在实验室的“我的虚拟机”列表中选择环境，查看和管理该环境中的资源。 
* 浏览 [Azure 快速入门模板库中的 Azure 资源管理器模板](https://github.com/Azure/azure-quickstart-templates)
