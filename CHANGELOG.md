# 修改记录

> 这个项目**没有启用 git**，也没有任何版本控制。
> 所以用这份文件手工记录所有改动：改了什么、为什么改、怎么还原。
> 每次修改都往下面追加一节，不要覆盖旧的。

---

## 2026-10-07 14:59 — 单人模式改用 Netcode 加载场景

**改动人**：Codex（AI 助手）

**文件**：`Assets/Scripts/KitchenGameMultiplayer.cs`

**位置**：第 47 行，`Start()` 方法里的单人分支

**改动内容**：

```diff
     private void Start() {
         if (!isMultiplayer) {
             //single player
             StartHost();
-            Loader.Load(Loader.Scene.GameScene);
+            Loader.LoadNetwork(Loader.Scene.GameScene);
         }
     }
```

**为什么要改**：

这个项目的 NetworkManager 开了 `EnableSceneManagement: 1`，在这种配置下
Netcode 只会为自己加载的场景生成场景内的 NetworkObject。

- 多人路径用的是 `Loader.LoadNetwork(...)`（Netcode 的场景管理），正常
- 单人路径原来用的是 `Loader.Load(...)`（普通 `SceneManager.LoadScene`），绕过了 Netcode

结果就是：单人进 GameScene 后，场景里的 NetworkObject（KitchenGameManger、
各个台面等）全部处于未 spawn 状态，`OnNetworkSpawn()` 一次都没跑过，
插槽表也没有初始化，于是按 E 就抛：

```
KeyNotFoundException: The given key 'KitchenGameManger' was not present in the dictionary.
  at Unity.Netcode.NetworkBehaviour.__endSendServerRpc (NetworkBehaviour.cs:134)
```

**判定依据**：日志里从未出现过 `[KitchenGameManager] Network spawned.`
（来自 `KitchenGameManger.cs:60` 的 `OnNetworkSpawn`）。

**副作用（要知道）**：单人模式不再经过 LoadingScene，会从 LobbyScene
直接切到 GameScene。这与作者在多人大厅、角色选择里的做法一致——那两处
也都是直接 `LoadNetwork`，同样没有加载场景。

**怎么还原**：

把第 47 行改回：

```csharp
Loader.Load(Loader.Scene.GameScene);
```

**验证状态**：⚠️ 尚未验证。改完后还没跑过一次。

---

## 2026-10-08 — 项目身份改成自己的

**改动人**：Codex（AI 助手）

**前提**：改动前 Unity 已关闭，并把要动的 76 个文件备份到
`_rename_backup_2026-10-08/`（74 个 .cs + ProjectSettings.asset + 旧 .slnx）。
此时项目没有版本控制，所以用文件夹备份兜底。**确认无误后可删除该备份目录。**

**改动内容**：

`ProjectSettings/ProjectSettings.asset`：

| 行 | 原值 | 新值 |
|---|---|---|
| 15 | `companyName: Kick Start` | `companyName: nemunama01` |
| 16 | `productName: Kitchen Chaos` | `productName: KitchenCooked` |
| 169 | `Standalone: com.DefaultCompany.KitchenChaos` | `Standalone: com.nemunama01.kitchencooked` |
| 890 | `metroPackageName: KitchenChaos` | `metroPackageName: KitchenCooked` |
| 897 | `metroApplicationDescription: KitchenChaos` | `metroApplicationDescription: KitchenCooked` |

源码署名：**74 个** `.cs` 文件里的 `Author: Bharath Kumar S`
全部改为 `Author: dm`。

文件名：`Kitchen-Chaos-main.slnx` → `KitchenCooked.slnx`。

**怎么还原**：从 `_rename_backup_2026-10-08/` 按相对路径拷回对应文件。
旧 .slnx 拷回后要自己改回文件名。

**未做**：项目**文件夹**本身仍叫 `Kitchen-Chaos-main`。见下方说明。

**验证状态**：⚠️ 还没在 Unity 里打开验证过。

---

## 项目环境状态（记录用，非改动）

| 项目 | 值 |
|---|---|
| Unity 版本 | 6000.2.13f1（`W:\DownloadApp\u3d\6000.2.13f1`） |
| Netcode 版本 | **2.1.1** |
| README 要求的 Netcode | **1.2.0**（作者明确警告不要升级） |

### Unity Cloud 关联（已配置）

`ProjectSettings/ProjectSettings.asset` 第 969 行起：

```
cloudProjectId: 1ba1b516-b7cd-4fe0-8152-0285a572cf0f
projectName: KitchenCooked
organizationId: nemunama01
```

原来的值是作者的 `42daf0d7-a54d-4f4f-a7d3-80a26ed77de9`，已替换。

**服务验证情况**：

- Authentication（匿名登录）：✅ 已验证可用（`SetPlayerIdServerRpc` 的空引用异常消失）
- Lobby：❓ 未验证（单人模式不经过）
- Relay：❓ 未验证（单人模式不经过）

---

## 关于"降级到 Netcode 1.2.0"

**结论：不可行，不推荐尝试。**

查过 Unity 官方包注册表后的数据：

| Netcode 版本 | 最低 Unity |
|---|---|
| 1.2.0 | 2020.3 |
| 1.15.1（1.x 最后一版） | 2022.3 |
| 2.1.1（当前） | 6000.0 |

三个阻塞点：

1. 整个 1.x 系列没有任何版本验证过 Unity 6。
2. 项目已绑定 Unity 6：URP 是 17.2.0（Unity 6 专属），Cinemachine 3.1.2。
   要回到 1.2.0 那个年代，等于把整个渲染管线重做。
3. 场景已被 2.x 重新序列化：NetworkManager 的网络预制体存在 `OldPrefabList`
   字段里，而 1.x 认的字段名是 `Prefabs`。降级会丢失整个网络预制体列表
   （共 12 个预制体）。

---

## 已处理的作者痕迹

以下原属于原作者，**已于 2026-10-08 全部改掉**：

- `companyName`：`Kick Start` → `nemunama01`
- `productName`：`Kitchen Chaos` → `KitchenCooked`
- `applicationIdentifier`：`com.DefaultCompany.KitchenChaos` → `com.nemunama01.kitchencooked`
- `metroPackageName` / `metroApplicationDescription` → `KitchenCooked`
- 74 个 `.cs` 文件头部的 `Author: Bharath Kumar S` → `dm`
- `Kitchen-Chaos-main.slnx` → `KitchenCooked.slnx`

**仍未处理**：

- 项目文件夹名还是 `Kitchen-Chaos-main`（原因见下）
- `README.md` 是作者的课程笔记，377 行，内容未动
- `UserSettings/Layouts/default-6000.dwlt` 里记着旧的项目路径，
  改文件夹名后 Unity 会自动更新，不用管

---

## 项目文件夹改名

文件夹从 `Kitchen-Chaos-main` 改成 `KitchenCooked` 需要手动做，原因是：

1. 这个文件夹是 AI 助手当前会话的工作根目录，改名会让它立刻失去访问权；
2. 重命名目录需要写父目录 `W:\File\U3dProject` 的权限，超出助手的沙箱范围。

步骤（Unity 和 AI 会话都关掉之后）：

```powershell
Rename-Item "W:\File\U3dProject\Kitchen-Chaos-main" "KitchenCooked"
```

`_rename_backup_2026-10-08/` 这个备份目录会跟着一起移动，不影响。

---

## 关于版本控制（重要，实测结论）

原本建议 `git init`，但实测发现一个坑：

**在这个项目里创建 `.git` 会让 AI 助手的全部命令失效。**

原因：助手的沙箱权限清单里 `…\Kitchen-Chaos-main\.git` 被单独列为
**只读**，且排在"项目根目录可写"之后。`.git` 一旦存在，这条规则开始生效，
与上一条冲突，沙箱初始化就报：

```
helper_unknown_error: setup refresh had errors
```

删掉 `.git` 后立刻恢复（已实测验证：删除前连 `echo` 都失败，删除后正常）。

所以现在的状态：

- 项目**没有** git 仓库
- 备份靠 `_rename_backup_2026-10-08/` 这种手动目录
- 如果要上版本控制，建议在自己的终端里操作，并接受
  「仓库存在时 AI 助手无法工作」这个限制