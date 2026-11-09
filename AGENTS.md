# AGENTS.md —— LenoWorkBuddy 的默认写法

> 给 AI / 新加入的人。**新代码照这 5 条写**；旧写法见文末"遗留（不要扩散）"。
> 完整论证见 `Leno/docs/idioms.md`。

| 场景 | 用这个 | 不要 |
|---|---|---|
| 数据 | `struct` + `Array[struct]` | `Array[Array[string]]` + `CUR bounds` 检查（`wb_gui.leno` 的 `ACCOUNTS`）|
| 状态 | `enum { ok = 0 ... }` | 字符串错误码散落各处（已有 `AUTH_ERROR`/`HELPER_TIMEOUT`…，**沿用别新增**）|
| 分支 | ≥3 个同值比较用 `switch` | 长 `if - else if` 链（`wb_auth.reason_text` 已改 `switch`）|
| 集合变换 | `.map` / `.filter` | `Array[x] out = []` + `for` + `add` |
| 错误 | 已有的 `Num` / 字符串错误码**继续用**，别再造第二种 | `catch e` 只 `print` |

已经存在、不要手写的（标准库）：

```leno
strings.pad_start(_str(v), 2, "0")   // 补零
strings.split(s, ",") / strings.lines(s)
strings.fmt_size(1536)               // "1 KB"
arr.sort(func(int a, int b) { return b - a })   // 自定义比较器（稳定）
jsons.encode(obj)                    // 生成 JSON，别手拼
```

## 遗留（不要扩散）

- `wb_growth.leno` 里自建的 JSON 取值 helper —— 优先用 `jsons.dig` / `jsons.get_str`
- `wb_gui.leno` 的 `ACCOUNTS` 三列数组 —— 新表用 `struct`
- 错误码字符串**已经成型**（`AUTH_ERROR` / `NO_SESSION` / `HELPER_*`），
  改的话先看 `wb_auth.reason_text`（那里是唯一的"代号 → 人话"表）

## 自检

```powershell
..\Leno\build\leno.exe --check wb_xxx.leno            # 单文件
Get-ChildItem test -Filter *.leno | % { ..\Leno\build\leno.exe $_.FullName }   # 6 个测试
..\Leno\build\leno.exe wb_sign.leno doctor            # 环境自检（doctor 子命令）
```
