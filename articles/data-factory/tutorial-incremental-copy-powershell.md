---
title: "使用 Azure 数据工厂以增量方式复制表 | Microsoft Docs"
description: "在本教程中，我们将创建一个 Azure 数据工厂管道，它能够以增量方式将 Azure SQL 数据库中的数据复制到 Azure Blob 存储。"
services: data-factory
documentationcenter: 
author: sharonlo101
manager: jhubbard
editor: spelluru
ms.service: data-factory
ms.workload: data-services
ms.tgt_pltfrm: na
ms.devlang: na
ms.topic: get-started-article
ms.date: 10/06/2017
ms.author: shlo
ms.openlocfilehash: 0b05971b5ab8ec3fd14dd4ce14d07df478e1dcc9
ms.sourcegitcommit: 5d3e99478a5f26e92d1e7f3cec6b0ff5fbd7cedf
ms.translationtype: HT
ms.contentlocale: zh-CN
ms.lasthandoff: 12/06/2017
---
# <a name="incrementally-load-data-from-azure-sql-database-to-azure-blob-storage"></a>以增量方式将 Azure SQL 数据库中的数据加载到 Azure Blob 存储
在本教程中，请创建一个带管道的 Azure 数据工厂，将增量数据从 Azure SQL 数据库中的表加载到 Azure Blob 存储。 


> [!NOTE]
> 本文适用于目前处于预览状态的数据工厂版本 2。 如果使用数据工厂服务版本 1（即正式版 (GA)），请参阅[数据工厂版本 1 文档](v1/data-factory-copy-data-from-azure-blob-storage-to-sql-database.md)。


在本教程中执行以下步骤：

> [!div class="checklist"]
> * 准备用于存储水印值的数据存储。   
> * 创建数据工厂。
> * 创建链接服务。 
> * 创建源、接收器和水印数据集。
> * 创建管道。
> * 运行管道。
> * 监视管道运行。 

## <a name="overview"></a>概述
高级解决方案示意图： 

![以增量方式加载数据](media\tutorial-Incrementally-copy-powershell\incrementally-load.png)

下面是创建此解决方案所要执行的重要步骤： 

1. **选择水印列**。
    在源数据存储中选择一个列，该列可用于将每个运行的新记录或已更新记录切片。 通常，在创建或更新行时，此选定列中的数据（例如 last_modify_time 或 ID）会不断递增。 此列中的最大值用作水印。
2. **准备用于存储水印值的数据存储**。   
    本教程在 Azure SQL 数据库中存储水印值。
3. **创建采用以下工作流的管道：** 
    
    此解决方案中的管道具有以下活动：
  
    1. 创建两个**查找**活动。 使用第一个查找活动检索最后一个水印值。 使用第二个查找活动检索新的水印值。 这些水印值会传递到复制活动。 
    2. 创建**复制活动**，用于复制源数据存储中其水印列值大于旧水印值且小于新水印值的行。 然后，该活动将源数据存储中的增量数据作为新文件复制到 Blob 存储。 
    3. 创建**存储过程活动**，用于更新下一次管道运行的水印值。 


如果你还没有 Azure 订阅，可以在开始前创建一个[免费](https://azure.microsoft.com/free/)帐户。

## <a name="prerequisites"></a>先决条件
* **Azure SQL 数据库**。 将数据库用作**源**数据存储。 如果没有 Azure SQL 数据库，请参阅[创建 Azure SQL 数据库](../sql-database/sql-database-get-started-portal.md)一文获取创建步骤。
* **Azure 存储帐户**。 将 Blob 存储用作**接收器**数据存储。 如果没有 Azure 存储帐户，请参阅[创建存储帐户](../storage/common/storage-create-storage-account.md#create-a-storage-account)一文获取创建步骤。 创建名为 **adftutorial** 的容器。 
* **Azure PowerShell**。 遵循[如何安装和配置 Azure PowerShell](/powershell/azure/install-azurerm-ps) 中的说明。

### <a name="create-a-data-source-table-in-your-azure-sql-database"></a>在 Azure SQL 数据库中创建数据源表
1. 打开 **SQL Server Management Studio**。 在“服务器资源管理器”中，右键单击数据库，然后选择“新建查询”。
2. 针对 Azure SQL 数据库运行以下 SQL 命令，创建名为 `data_source_table` 的表作为数据源存储。  
    
    ```sql
    create table data_source_table
    (
        PersonID int,
        Name varchar(255),
        LastModifytime datetime
    );

    INSERT INTO data_source_table
    (PersonID, Name, LastModifytime)
    VALUES
    (1, 'aaaa','9/1/2017 12:56:00 AM'),
    (2, 'bbbb','9/2/2017 5:23:00 AM'),
    (3, 'cccc','9/3/2017 2:36:00 AM'),
    (4, 'dddd','9/4/2017 3:21:00 AM'),
    (5, 'eeee','9/5/2017 8:06:00 AM');
    ```
    本教程使用 **LastModifytime** 作为**水印**列。  下表显示了数据源存储中的数据：

    ```
    PersonID | Name | LastModifytime
    -------- | ---- | --------------
    1 | aaaa | 2017-09-01 00:56:00.000
    2 | bbbb | 2017-09-02 05:23:00.000
    3 | cccc | 2017-09-03 02:36:00.000
    4 | dddd | 2017-09-04 03:21:00.000
    5 | eeee | 2017-09-05 08:06:00.000
    ```

### <a name="create-another-table-in-sql-database-to-store-the-high-watermark-value"></a>在 SQL 数据库中创建另一个表用于存储高水印值
1. 针对 Azure SQL 数据库运行以下 SQL 命令，创建名为 `watermarktable` 的表用于存储水印值。  
    
    ```sql
    create table watermarktable
    (
    
    TableName varchar(255),
    WatermarkValue datetime,
    );
    ```
3. 使用源数据存储的表名设置高水印的默认**值**。  （在本教程中，表名为：**data_source_table**）

    ```sql
    INSERT INTO watermarktable
    VALUES ('data_source_table','1/1/2010 12:00:00 AM')    
    ```
4. 查看表中的数据：`watermarktable`。
    
    ```sql
    Select * from watermarktable
    ```
    输出： 

    ```
    TableName  | WatermarkValue
    ----------  | --------------
    data_source_table | 2010-01-01 00:00:00.000
    ```

### <a name="create-a-stored-procedure-in-azure-sql-database"></a>在 Azure SQL 数据库中创建存储过程 

运行以下命令，在 Azure SQL 数据库中创建存储过程。

```sql
CREATE PROCEDURE sp_write_watermark @LastModifiedtime datetime, @TableName varchar(50)
AS

BEGIN
    
    UPDATE watermarktable
    SET [WatermarkValue] = @LastModifiedtime 
WHERE [TableName] = @TableName
    
END
```

## <a name="create-a-data-factory"></a>创建数据工厂
1. 为资源组名称定义一个变量，稍后会在 PowerShell 命令中使用该变量。 将以下命令文本复制到 PowerShell，在双引号中指定 [Azure 资源组](../azure-resource-manager/resource-group-overview.md)的名称，然后运行命令。 例如：`"adfrg"`。 
   
     ```powershell
    $resourceGroupName = "ADFTutorialResourceGroup";
    ```

    如果该资源组已存在，请勿覆盖它。 为 `$resourceGroupName` 变量分配另一个值，然后再次运行命令
2. 定义一个用于数据工厂位置的变量： 

    ```powershell
    $location = "East US"
    ```
3. 若要创建 Azure 资源组，请运行以下命令： 

    ```powershell
    New-AzureRmResourceGroup $resourceGroupName $location
    ``` 
    如果该资源组已存在，请勿覆盖它。 为 `$resourceGroupName` 变量分配另一个值，然后再次运行命令。 
3. 定义一个用于数据工厂名称的变量。 

    > [!IMPORTANT]
    >  更新数据工厂名称，使之全局唯一。 例如 ADFTutorialFactorySP1127。 

    ```powershell
    $dataFactoryName = "ADFIncCopyTutorialFactory";
    ```
5. 若要创建数据工厂，请运行以下 **Set-AzureRmDataFactoryV2** cmdlet： 
    
    ```powershell       
    Set-AzureRmDataFactoryV2 -ResourceGroupName $resourceGroupName -Location "East US" -Name $dataFactoryName 
    ```

请注意以下几点：

* Azure 数据工厂的名称必须全局唯一。 如果收到以下错误，请更改名称并重试。

    ```
    The specified Data Factory name 'ADFv2QuickStartDataFactory' is already in use. Data Factory names must be globally unique.
    ```
* 若要创建数据工厂实例，用于登录到 Azure 的用户帐户必须属于**参与者**或**所有者**角色，或者是 Azure 订阅的**管理员**。
* 目前，数据工厂版本 2 仅允许在“美国东部”、“美国东部 2”和“西欧”区域创建数据工厂。 数据工厂使用的数据存储（Azure 存储、Azure SQL 数据库，等等）和计算资源（HDInsight 等）可以位于其他区域中。


## <a name="create-linked-services"></a>创建链接服务
可在数据工厂中创建链接服务，将数据存储和计算服务链接到数据工厂。 在本部分中，创建 Azure 存储帐户和 Azure SQL 数据库的链接服务。 

### <a name="create-azure-storage-linked-service"></a>创建 Azure 存储链接服务。
1. 在 **C:\ADF** 文件夹中，创建包含以下内容的名为 **AzureStorageLinkedService.json** 的 JSON 文件：（如果文件夹 ADF 不存在，请创建该文件夹）。 将 `<accountName>` 和 `<accountKey>` 分别替换为 Azure 存储帐户的名称和密钥，然后保存文件。

    ```json
    {
        "name": "AzureStorageLinkedService",
        "properties": {
            "type": "AzureStorage",
            "typeProperties": {
                "connectionString": {
                    "value": "DefaultEndpointsProtocol=https;AccountName=<accountName>;AccountKey=<accountKey>",
                    "type": "SecureString"
                }
            }
        }
    }
    ```
2. 在 **Azure PowerShell** 中，切换到 **ADF** 文件夹。
3. 运行 **Set-AzureRmDataFactoryV2LinkedService** cmdlet 创建链接服务：**AzureStorageLinkedService**。 在以下示例中，传递 **ResourceGroupName** 和 **DataFactoryName** 参数的值。 

    ```powershell
    Set-AzureRmDataFactoryV2LinkedService -DataFactoryName $dataFactoryName -ResourceGroupName $resourceGroupName -Name "AzureStorageLinkedService" -File ".\AzureStorageLinkedService.json"
    ```

    下面是示例输出：

    ```json
    LinkedServiceName : AzureStorageLinkedService
    ResourceGroupName : <resourceGroupName>
    DataFactoryName   : <dataFactoryName>
    Properties        : Microsoft.Azure.Management.DataFactory.Models.AzureStorageLinkedService
    ```

### <a name="create-azure-sql-database-linked-service"></a>创建 Azure SQL 数据库链接服务
1. 在 **C:\ADF** 文件夹中，创建包含以下内容的名为 **AzureSQLDatabaseLinkedService.json** 的 JSON 文件：（如果文件夹 ADF 不存在，请创建该文件夹）。 将 **&lt;server&gt;、&lt;database&gt;、&lt;user id&gt; 和 &lt;password&gt;** 分别替换为自己的 Azure SQL Server 名称、用户 ID 和密码，然后保存文件。 

    ```json
    {
        "name": "AzureSQLDatabaseLinkedService",
        "properties": {
            "type": "AzureSqlDatabase",
            "typeProperties": {
                "connectionString": {
                    "value": "Server = tcp:<server>.database.windows.net,1433;Initial Catalog=<database>; Persist Security Info=False; User ID=<user> ; Password=<password>; MultipleActiveResultSets = False; Encrypt = True; TrustServerCertificate = False; Connection Timeout = 30;",
                    "type": "SecureString"
                }
            }
        }
    }
    ```
1. 在 **Azure PowerShell** 中，切换到 **ADF** 文件夹。
2. 运行 **Set-AzureRmDataFactoryV2LinkedService** cmdlet 创建链接服务：**AzureSQLDatabaseLinkedService**。 

    ```powershell
    Set-AzureRmDataFactoryV2LinkedService -DataFactoryName $dataFactoryName -ResourceGroupName $resourceGroupName -Name "AzureSQLDatabaseLinkedService" -File ".\AzureSQLDatabaseLinkedService.json"
    ```

    下面是示例输出：

    ```json
    LinkedServiceName : AzureSQLDatabaseLinkedService
    ResourceGroupName : ADF
    DataFactoryName   : incrementalloadingADF
    Properties        : Microsoft.Azure.Management.DataFactory.Models.AzureSqlDatabaseLinkedService
    ProvisioningState :
    ```

## <a name="create-datasets"></a>创建数据集
在此步骤中，创建表示源和接收器数据的数据集。 

### <a name="create-a-source-dataset"></a>创建源数据集

1. 在同一文件夹中，创建包含以下内容的名为 SourceDataset.json 的 JSON 文件： 

    ```json
    {
        "name": "SourceDataset",
        "properties": {
            "type": "AzureSqlTable",
            "typeProperties": {
                "tableName": "data_source_table"
            },
            "linkedServiceName": {
                "referenceName": "AzureSQLDatabaseLinkedService",
                "type": "LinkedServiceReference"
            }
        }
    }
   
    ```
    本教程使用表名 **data_source_table**。 如果使用具有不同名称的表，请替换名称。 
2.  运行 Set-AzureRmDataFactoryV2Dataset cmdlet 创建数据集：SourceDataset
    
    ```powershell
    Set-AzureRmDataFactoryV2Dataset -DataFactoryName $dataFactoryName -ResourceGroupName $resourceGroupName -Name "SourceDataset" -File ".\SourceDataset.json"
    ```

    下面是该 cmdlet 的示例输出：
    
    ```json
    DatasetName       : SourceDataset
    ResourceGroupName : ADF
    DataFactoryName   : incrementalloadingADF
    Structure         :
    Properties        : Microsoft.Azure.Management.DataFactory.Models.AzureSqlTableDataset
    ```

### <a name="create-a-sink-dataset"></a>创建接收器数据集

1. 在同一文件夹中，创建包含以下内容的名为 SinkDataset.json 的 JSON 文件： 

    ```json
    {
        "name": "SinkDataset",
        "properties": {
            "type": "AzureBlob",
            "typeProperties": {
                "folderPath": "adftutorial/incrementalcopy",
                "fileName": "@CONCAT('Incremental-', pipeline().RunId, '.txt')", 
                "format": {
                    "type": "TextFormat"
                }
            },
            "linkedServiceName": {
                "referenceName": "AzureStorageLinkedService",
                "type": "LinkedServiceReference"
            }
        }
    }   
    ```

    > [!IMPORTANT]
    > 此代码片段假设 Azure Blob 存储中有一个名为 **adftutorial** 的 Blob 容器。 创建容器（如果不存在），或者将容器设置为现有容器的名称。 会自动创建输出文件夹 `incrementalcopy`（如果容器中不存在）。 在本教程中，文件名是使用以下表达式动态生成的：`@CONCAT('Incremental-', pipeline().RunId, '.txt')`。
2.  运行 Set-AzureRmDataFactoryV2Dataset cmdlet 创建数据集：SinkDataset
    
    ```powershell
    Set-AzureRmDataFactoryV2Dataset -DataFactoryName $dataFactoryName -ResourceGroupName $resourceGroupName -Name "SinkDataset" -File ".\SinkDataset.json"
    ```

    下面是该 cmdlet 的示例输出：
    
    ```json
    DatasetName       : SinkDataset
    ResourceGroupName : ADF
    DataFactoryName   : incrementalloadingADF
    Structure         :
    Properties        : Microsoft.Azure.Management.DataFactory.Models.AzureBlobDataset    
    ```

## <a name="create-a-dataset-for-watermark"></a>为水印创建数据集
在此步骤中，创建用于存储高水印值的数据集。 

1. 在同一文件夹中，创建包含以下内容的名为 WatermarkDataset.json 的 JSON 文件： 

    ```json
    {
        "name": " WatermarkDataset ",
        "properties": {
            "type": "AzureSqlTable",
            "typeProperties": {
                "tableName": "watermarktable"
            },
            "linkedServiceName": {
                "referenceName": "AzureSQLDatabaseLinkedService",
                "type": "LinkedServiceReference"
            }
        }
    }    
    ```
2.  运行 Set-AzureRmDataFactoryV2Dataset cmdlet 创建数据集：WatermarkDataset
    
    ```powershell
    Set-AzureRmDataFactoryV2Dataset -DataFactoryName $dataFactoryName -ResourceGroupName $resourceGroupName -Name "WatermarkDataset" -File ".\WatermarkDataset.json"
    ```

    下面是该 cmdlet 的示例输出：
    
    ```json
    DatasetName       : WatermarkDataset
    ResourceGroupName : ADF
    DataFactoryName   : incrementalloadingADF
    Structure         :
    Properties        : Microsoft.Azure.Management.DataFactory.Models.AzureSqlTableDataset    
    ```

## <a name="create-a-pipeline"></a>创建管道
本教程创建包含两个查找活动、一个复制活动和存储过程活动的管道，这些活动链接在一个管道中。 


1. 在同一文件夹中，创建包含以下内容的 JSON 文件 IncrementalCopyPipeline.json： 

    ```json
    {
        "name": "IncrementalCopyPipeline",
        "properties": {
            "activities": [
                {
                    "name": "LookupOldWaterMarkActivity",
                    "type": "Lookup",
                    "typeProperties": {
                        "source": {
                        "type": "SqlSource",
                        "sqlReaderQuery": "select * from watermarktable"
                        },
    
                        "dataset": {
                        "referenceName": "WatermarkDataset",
                        "type": "DatasetReference"
                        }
                    }
                },
                {
                    "name": "LookupNewWaterMarkActivity",
                    "type": "Lookup",
                    "typeProperties": {
                        "source": {
                            "type": "SqlSource",
                            "sqlReaderQuery": "select MAX(LastModifytime) as NewWatermarkvalue from data_source_table"
                        },
    
                        "dataset": {
                        "referenceName": "SourceDataset",
                        "type": "DatasetReference"
                        }
                    }
                },
                
                {
                    "name": "IncrementalCopyActivity",
                    "type": "Copy",
                    "typeProperties": {
                        "source": {
                            "type": "SqlSource",
                            "sqlReaderQuery": "select * from data_source_table where LastModifytime > '@{activity('LookupOldWaterMarkActivity').output.firstRow.WatermarkValue}' and LastModifytime <= '@{activity('LookupNewWaterMarkActivity').output.firstRow.NewWatermarkvalue}'"
                        },
                        "sink": {
                            "type": "BlobSink"
                        }
                    },
                    "dependsOn": [
                        {
                            "activity": "LookupNewWaterMarkActivity",
                            "dependencyConditions": [
                                "Succeeded"
                            ]
                        },
                        {
                            "activity": "LookupOldWaterMarkActivity",
                            "dependencyConditions": [
                                "Succeeded"
                            ]
                        }
                    ],
    
                    "inputs": [
                        {
                            "referenceName": "SourceDataset",
                            "type": "DatasetReference"
                        }
                    ],
                    "outputs": [
                        {
                            "referenceName": "SinkDataset",
                            "type": "DatasetReference"
                        }
                    ]
                },
    
                {
                    "name": "StoredProceduretoWriteWatermarkActivity",
                    "type": "SqlServerStoredProcedure",
                    "typeProperties": {
    
                        "storedProcedureName": "sp_write_watermark",
                        "storedProcedureParameters": {
                            "LastModifiedtime": {"value": "@{activity('LookupNewWaterMarkActivity').output.firstRow.NewWatermarkvalue}", "type": "datetime" },
                            "TableName":  { "value":"@{activity('LookupOldWaterMarkActivity').output.firstRow.TableName}", "type":"String"}
                        }
                    },
    
                    "linkedServiceName": {
                        "referenceName": "AzureSQLDatabaseLinkedService",
                        "type": "LinkedServiceReference"
                    },
    
                    "dependsOn": [
                        {
                            "activity": "IncrementalCopyActivity",
                            "dependencyConditions": [
                                "Succeeded"
                            ]
                        }
                    ]
                }
            ]
            
        }
    }
    ```
    

2. 运行 Set-AzureRmDataFactoryV2Pipeline cmdlet，以便创建管道 IncrementalCopyPipeline。
    
   ```powershell
   Set-AzureRmDataFactoryV2Pipeline -DataFactoryName $dataFactoryName -ResourceGroupName $resourceGroupName -Name "IncrementalCopyPipeline" -File ".\IncrementalCopyPipeline.json"
   ``` 

   下面是示例输出： 

   ```json
    PipelineName      : IncrementalCopyPipeline
    ResourceGroupName : ADF
    DataFactoryName   : incrementalloadingADF
    Activities        : {LookupOldWaterMarkActivity, LookupNewWaterMarkActivity, IncrementalCopyActivity, StoredProceduretoWriteWatermarkActivity}
    Parameters        :
   ```
 
## <a name="run-the-pipeline"></a>运行管道

1. 使用 **Invoke-AzureRmDataFactoryV2Pipeline** cmdlet 运行管道：**IncrementalCopyPipeline**。 将占位符替换为自己的资源组和数据工厂名称。

    ```powershell
    $RunId = Invoke-AzureRmDataFactoryV2Pipeline -PipelineName "IncrementalCopyPipeline" -ResourceGroupName $resourceGroupName -dataFactoryName $dataFactoryName
    ``` 
2. 运行 Get-AzureRmDataFactoryV2ActivityRun cmdlet 检查管道的状态，直到看到所有活动成功运行的消息。 将占位符替换为针对 RunStartedAfter 和 RunStartedBefore 参数指定的、自己的适当时间。  本教程使用 -RunStartedAfter "2017/09/14" -RunStartedBefore "2017/09/15"

    ```powershell
    Get-AzureRmDataFactoryV2ActivityRun -DataFactoryName $dataFactoryName -ResourceGroupName $resourceGroupName -PipelineRunId $RunId -RunStartedAfter "<start time>" -RunStartedBefore "<end time>"
    ```

    下面是示例输出：
 
    ```json
    ResourceGroupName : ADF
    DataFactoryName   : incrementalloadingADF
    ActivityName      : LookupNewWaterMarkActivity
    PipelineRunId     : d4bf3ce2-5d60-43f3-9318-923155f61037
    PipelineName      : IncrementalCopyPipeline
    Input             : {source, dataset}
    Output            : {NewWatermarkvalue}
    LinkedServiceName :
    ActivityRunStart  : 9/14/2017 7:42:42 AM
    ActivityRunEnd    : 9/14/2017 7:42:50 AM
    DurationInMs      : 7777
    Status            : Succeeded
    Error             : {errorCode, message, failureType, target}
    
    ResourceGroupName : ADF
    DataFactoryName   : incrementalloadingADF
    ActivityName      : LookupOldWaterMarkActivity
    PipelineRunId     : d4bf3ce2-5d60-43f3-9318-923155f61037
    PipelineName      : IncrementalCopyPipeline
    Input             : {source, dataset}
    Output            : {TableName, WatermarkValue}
    LinkedServiceName :
    ActivityRunStart  : 9/14/2017 7:42:42 AM
    ActivityRunEnd    : 9/14/2017 7:43:07 AM
    DurationInMs      : 25437
    Status            : Succeeded
    Error             : {errorCode, message, failureType, target}
    
    ResourceGroupName : ADF
    DataFactoryName   : incrementalloadingADF
    ActivityName      : IncrementalCopyActivity
    PipelineRunId     : d4bf3ce2-5d60-43f3-9318-923155f61037
    PipelineName      : IncrementalCopyPipeline
    Input             : {source, sink}
    Output            : {dataRead, dataWritten, rowsCopied, copyDuration...}
    LinkedServiceName :
    ActivityRunStart  : 9/14/2017 7:43:10 AM
    ActivityRunEnd    : 9/14/2017 7:43:29 AM
    DurationInMs      : 19769
    Status            : Succeeded
    Error             : {errorCode, message, failureType, target}
    
    ResourceGroupName : ADF
    DataFactoryName   : incrementalloadingADF
    ActivityName      : StoredProceduretoWriteWatermarkActivity
    PipelineRunId     : d4bf3ce2-5d60-43f3-9318-923155f61037
    PipelineName      : IncrementalCopyPipeline
    Input             : {storedProcedureName, storedProcedureParameters}
    Output            : {}
    LinkedServiceName :
    ActivityRunStart  : 9/14/2017 7:43:32 AM
    ActivityRunEnd    : 9/14/2017 7:43:47 AM
    DurationInMs      : 14467
    Status            : Succeeded
    Error             : {errorCode, message, failureType, target}

    ```

## <a name="review-the-results"></a>查看结果

1. 在 Azure Blob 存储（接收器存储）中，应会看到数据已复制到 SinkDataset 中定义的文件。  在当前的教程中，文件名为 `Incremental- d4bf3ce2-5d60-43f3-9318-923155f61037.txt`。  打开该文件，可以看到其中与 Azure SQL 数据库中的数据相同的记录。

    ```
    1,aaaa,2017-09-01 00:56:00.0000000
    2,bbbb,2017-09-02 05:23:00.0000000
    3,cccc,2017-09-03 02:36:00.0000000
    4,dddd,2017-09-04 03:21:00.0000000
    5,eeee,2017-09-05 08:06:00.0000000
    ``` 
2. 检查 `watermarktable` 中的最新值，会看到水印值已更新。

    ```sql
    Select * from watermarktable
    ```
    
    下面是示例输出：
 
    TableName | WatermarkValue
    --------- | --------------
    data_source_table   2017-09-05  8:06:00.000

### <a name="insert-data-into-data-source-store-to-verify-delta-data-loading"></a>将数据插入数据源存储，验证增量数据的加载

1. 在 Azure SQL 数据库（数据源存储）中插入新数据：

    ```sql
    INSERT INTO data_source_table
    VALUES (6, 'newdata','9/6/2017 2:23:00 AM')
    
    INSERT INTO data_source_table
    VALUES (7, 'newdata','9/7/2017 9:01:00 AM')
    ``` 

    Azure SQL 数据库中的更新数据为：

    ```
    PersonID | Name | LastModifytime
    -------- | ---- | --------------
    1 | aaaa | 2017-09-01 00:56:00.000
    2 | bbbb | 2017-09-02 05:23:00.000
    3 | cccc | 2017-09-03 02:36:00.000
    4 | dddd | 2017-09-04 03:21:00.000
    5 | eeee | 2017-09-05 08:06:00.000
    6 | newdata | 2017-09-06 02:23:00.000
    7 | newdata | 2017-09-07 09:01:00.000
    ```
2. 再次使用 **Invoke-AzureRmDataFactoryV2Pipeline** cmdlet 运行管道：**IncrementalCopyPipeline**。 将占位符替换为自己的资源组和数据工厂名称。

    ```powershell
    $RunId = Invoke-AzureRmDataFactoryV2Pipeline -PipelineName "IncrementalCopyPipeline" -ResourceGroupName $resourceGroupName -dataFactoryName $dataFactoryName
    ```
3. 运行 **Get-AzureRmDataFactoryV2ActivityRun** cmdlet 检查管道的状态，直到看到所有活动成功运行的消息。 将占位符替换为针对 RunStartedAfter 和 RunStartedBefore 参数指定的、自己的适当时间。  本教程使用 -RunStartedAfter "2017/09/14" -RunStartedBefore "2017/09/15"

    ```powershell
    Get-AzureRmDataFactoryV2ActivityRun -DataFactoryName $dataFactoryName -ResourceGroupName $resourceGroupName -PipelineRunId $RunId -RunStartedAfter "<start time>" -RunStartedBefore "<end time>"
    ```

    下面是示例输出：
 
    ```json
    ResourceGroupName : ADF
    DataFactoryName   : incrementalloadingADF
    ActivityName      : LookupNewWaterMarkActivity
    PipelineRunId     : 2fc90ab8-d42c-4583-aa64-755dba9925d7
    PipelineName      : IncrementalCopyPipeline
    Input             : {source, dataset}
    Output            : {NewWatermarkvalue}
    LinkedServiceName :
    ActivityRunStart  : 9/14/2017 8:52:26 AM
    ActivityRunEnd    : 9/14/2017 8:52:58 AM
    DurationInMs      : 31758
    Status            : Succeeded
    Error             : {errorCode, message, failureType, target}
    
    ResourceGroupName : ADF
    DataFactoryName   : incrementalloadingADF
    ActivityName      : LookupOldWaterMarkActivity
    PipelineRunId     : 2fc90ab8-d42c-4583-aa64-755dba9925d7
    PipelineName      : IncrementalCopyPipeline
    Input             : {source, dataset}
    Output            : {TableName, WatermarkValue}
    LinkedServiceName :
    ActivityRunStart  : 9/14/2017 8:52:26 AM
    ActivityRunEnd    : 9/14/2017 8:52:52 AM
    DurationInMs      : 25497
    Status            : Succeeded
    Error             : {errorCode, message, failureType, target}
    
    ResourceGroupName : ADF
    DataFactoryName   : incrementalloadingADF
    ActivityName      : IncrementalCopyActivity
    PipelineRunId     : 2fc90ab8-d42c-4583-aa64-755dba9925d7
    PipelineName      : IncrementalCopyPipeline
    Input             : {source, sink}
    Output            : {dataRead, dataWritten, rowsCopied, copyDuration...}
    LinkedServiceName :
    ActivityRunStart  : 9/14/2017 8:53:00 AM
    ActivityRunEnd    : 9/14/2017 8:53:20 AM
    DurationInMs      : 20194
    Status            : Succeeded
    Error             : {errorCode, message, failureType, target}
    
    ResourceGroupName : ADF
    DataFactoryName   : incrementalloadingADF
    ActivityName      : StoredProceduretoWriteWatermarkActivity
    PipelineRunId     : 2fc90ab8-d42c-4583-aa64-755dba9925d7
    PipelineName      : IncrementalCopyPipeline
    Input             : {storedProcedureName, storedProcedureParameters}
    Output            : {}
    LinkedServiceName :
    ActivityRunStart  : 9/14/2017 8:53:23 AM
    ActivityRunEnd    : 9/14/2017 8:53:41 AM
    DurationInMs      : 18502
    Status            : Succeeded
    Error             : {errorCode, message, failureType, target}

    ```
4.  在 Azure Blob 存储中，应会看到已在 Azure Blob 存储中创建了另一个文件。 在本教程中，新文件名为 `Incremental-2fc90ab8-d42c-4583-aa64-755dba9925d7.txt`。  打开该文件，会看到其中包含两行记录：
5.  检查 `watermarktable` 中的最新值，会看到水印值已再次更新

    ```sql
    Select * from watermarktable
    ```
    示例输出： 
    
    TableName | WatermarkValue
    --------- | ---------------
    data_source_table | 2017-09-07 09:01:00.000

     
## <a name="next-steps"></a>后续步骤
已在本教程中执行了以下步骤： 

> [!div class="checklist"]
> * 定义**水印**列并将其存储在 Azure SQL 数据库中。  
> * 创建数据工厂。
> * 创建 SQL 数据库和 Blob 存储的链接服务。 
> * 创建源和接收器数据集。
> * 创建管道。
> * 运行管道。
> * 监视管道运行。 

在本教程中，管道将数据从 Azure SQL 数据库中的**单个表**复制到了 Azure Blob 存储。 转到下面的教程，了解如何将数据从本地 SQL Server 数据库中的**多个表**复制到 Azure SQL 数据库。 

> [!div class="nextstepaction"]
>[以增量方式将数据从 SQL Server 中的多个表加载到 Azure SQL 数据库](tutorial-incremental-copy-multiple-tables-powershell.md)



