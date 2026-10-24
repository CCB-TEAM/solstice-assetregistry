# solstice-assetregistry

`AssetRegistry.bin`（UE v21 格式）的读取与**追加**库。

上游 [CCB-TEAM/UAssetRegistry](https://github.com/CCB-TEAM/UAssetRegistry) 提供了读取能力；本仓库把它编成库，并补上「不改动既有数据、只在末尾追加资产」所需的两个补丁。

## 为什么要打补丁

直接编译上游源码（`external/UAssetRegistry/AssetRegistry/**`）能读，但写出时有两处会坏：

| 文件 | 问题 | 处理 |
|---|---|---|
| `FAssetRegistryWriter.cs` | tag map handle 的 bit63 处理与 v21 不符 | 换成 `Vendored/` 下的副本 |
| `FAssetData.cs` | `FTopLevelAssetPath` 少了 `ClassPackage` 段 | 同上 |

所以 csproj 里 `Compile Remove` 掉上游那两份，改用 `Vendored/` 的补丁版。

## 追加策略

`NameBatchRebuilder` 负责在**不动原有 name batch** 的前提下追加新名字：

- 解析 `count / numStringBytes / hashVersion(0xC1640000) / hashes[] / 2 字节头 / 字符串池`
- 新名字追加到池尾，哈希表按 UE 的 CityHash64 重新计算（复用 [CCB-TEAM/ULocres](https://github.com/CCB-TEAM/ULocres) 的实现，不自己造轮子）
- `FName` 是变长的：带编号的名字是 8 字节（bit 0x80000000）

## 用法

```bash
git clone --recursive https://github.com/CCB-TEAM/solstice-assetregistry.git
dotnet build src/Solstice.AssetRegistry
```

```csharp
using UAssetRegistry;

var state = AssetRegistryLoader.LoadFromFile("AssetRegistry.bin");
foreach (var asset in state.Assets)
{
    Console.WriteLine($"{asset.PackagePath} -> {asset.AssetClass}");
}
```

## 依赖

- `UAssetAPI` 1.1.0（NuGet）
- 子模块：`external/UAssetRegistry`、`external/ULocres`

克隆时务必带 `--recursive`。

## 验证过的行为

在真实 `AssetRegistry.bin`（6,459,113 字节 / 16,250 资产 / 30,417 名字）上：

- 解析出的 name batch 与 store 边界精确对齐（store 结束标记 `0x87654321` 落在预期位置）
- 追加 2 个卡牌资产后，重新序列化的 store 段与原文件**逐字节一致**（4,239,547 字节）
- 名字哈希 0 处不匹配

仓库里不含任何游戏资产。

## 限制

- 只验证过 v21
- 追加只能在末尾加新资产；删除或改动既有条目不在支持范围
- 不改动已存在的 name batch 内容，只追加

## 许可

MIT
