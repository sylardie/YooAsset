# 资源构建

![image](https://github.com/tuyoogame/YooAsset/raw/main/Docs/Image/AssetBuilder-img1.png)

### 界面介绍

- **Build Output**

  构建输出的目录，会根据Unity编辑器当前切换的平台自动划分构建结果。

- **Build Version**

  构建版本号，也是资源版本号，版本号必须大于零。

- **Build Pipeline**

  构建管线

  (1) BuiltinBuildPipeline: 传统的内置构建管线。

  (2) ScriptableBuildPipeline: 可编程构建管线。

- **Build Mode**

  构建模式

  (1) 强制构建模式：会删除指定构建平台下的所有构建记录，重新构建所有资源包。

  (2) 增量构建模式：以上一次构建结果为基础，对于发生变化的资源进行增量构建。

  (3) 演练构建模式：在不生成AssetBundle文件的前提下，进行演练构建并快速生成构建报告和补丁清单。

  (4) 模拟构建模式：在编辑器下配合EditorSimulateMode运行模式，来模拟真实运行的环境。

- **Encryption**

  加密类列表。

- **Compression**

  资源包的压缩方式。

- **Append Extension**

  构建的资源包文件名是否包含后缀格式。

- **Buildin Tags**

  标记为安装包里的资源标签列表。构建成功后，会将相关标记的资源包拷贝到StreamingAssets文件夹下。

- **构建**

  点击构建按钮会开始构建流程，构建流程分为多个节点顺序执行，如果某个节点发生错误，会导致构建失败。错误信息可以在控制台查看。

### 资源包加密

编写继承IEncryptionServices接口的加密类。注意：加密类文件需要放置在Editor文件夹里。

````C#
using System;
using YooAsset.Editor;

public class GameEncryption : IEncryptionServices
{
    /// <summary>
    /// 检测资源包是否需要加密
    /// </summary>
    bool IEncryptionServices.Check(string bundleName)
    {
        // 对配置表相关的资源包进行加密
        return bundleName.Contains("assets/config/");
    }

    /// <summary>
    /// 对数据进行加密，并返回加密后的数据
    /// </summary>
    byte[] IEncryptionServices.Encrypt(byte[] fileData)
    {
        int offset = 32;
        var temper = new byte[fileData.Length + offset];
        Buffer.BlockCopy(fileData, 0, temper, offset, fileData.Length);
        return temper;
    }
}
````

### 补丁包

构建成功后会在输出目录下找到补丁包文件夹，该文件夹名称为本次构建时指定的资源版本号。

补丁包文件夹里包含补丁清单文件，资源包文件，构建报告文件等。

资源包文件都是以文件的哈希值命名。

![image](https://github.com/tuyoogame/YooAsset/raw/main/Docs/Image/AssetBuilder-img4.png)

### 补丁清单

补丁清单是一个Json格式的文本文件，里面包含了所有资源包的信息，例如：名称，大小，CRC等。

![image](https://github.com/tuyoogame/YooAsset/raw/main/Docs/Image/AssetBuilder-img2.png)

### Jenkins支持

如果需要自动化构建，可以参考如下代码范例：

使用内置构建管线来构建资源包。

````c#
private static void BuildInternal(BuildTarget buildTarget)
{
    Debug.Log($"开始构建 : {buildTarget}");

    // 命令行参数
    int buildVersion = GetBuildVersion();

    // 构建参数
    string defaultOutputRoot = AssetBundleBuilderHelper.GetDefaultOutputRoot();
    BuildParameters buildParameters = new BuildParameters();
    buildParameters.OutputRoot = defaultOutputRoot;
    buildParameters.BuildTarget = buildTarget;
    buildParameters.BuildPipeline = EBuildPipeline.BuiltinBuildPipeline;
    buildParameters.BuildMode = EBuildMode.ForceRebuild;
    buildParameters.BuildVersion = buildVersion;
    buildParameters.BuildinTags = "buildin";
    buildParameters.VerifyBuildingResult = true;
    buildParameters.EnableAddressable = false;
    buildParameters.AppendFileExtension = false;
    buildParameters.CopyBuildinTagFiles = true;
    buildParameters.EncryptionServices = new GameEncryption();
    buildParameters.CompressOption = ECompressOption.LZ4;
    
    // 执行构建
    AssetBundleBuilder builder = new AssetBundleBuilder();
    bool succeed = builder.Run(buildParameters);
    Debug.Log($"构建结果:{succeed}");
}

// 从构建命令里获取参数
private static int GetBuildVersion()
{
    foreach (string arg in System.Environment.GetCommandLineArgs())
    {
        if (arg.StartsWith("buildVersion"))
            return int.Parse(arg.Split("="[0])[1]);
    }
    return -1;
}
````

