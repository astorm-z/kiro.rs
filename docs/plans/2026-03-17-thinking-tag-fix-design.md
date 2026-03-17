# Thinking 标签处理修复设计文档

## 概述

修复 kiro.rs 项目中 thinking 标签处理的 bug，确保 `<thinking>` 和 `</thinking>` 标签被正确转换为 Anthropic API 的 reasoning_content 格式，不会泄漏到输出中。

## 背景

kiro.rs 已经实现了基本的 thinking 标签转换功能，将 Kiro API 返回的 `<thinking>...</thinking>` 内容转换为 Anthropic Messages API 的 `thinking` 块格式。但现有实现存在以下问题：

1. **标签泄漏风险**：`find_real_thinking_end_tag` 要求 `</thinking>` 后必须有 `\n\n`，但模型可能输出 `</thinking>\n` 或 `</thinking>` 后直接结束
2. **缓冲区处理不一致**：多处手动计算保留长度，容易出错
3. **边界情况覆盖不全**：某些边界情况未被正确处理

## 设计目标

- 修复标签泄漏问题，确保标签不会出现在最终输出中
- 统一缓冲区处理逻辑，减少重复代码
- 保持现有架构不变，最小化改动
- 改动量控制在 50-80 行代码

## 技术方案

### 1. 增强结束标签检测逻辑

**修改 `find_real_thinking_end_tag` 函数**：

- **原逻辑**：要求 `</thinking>` 后必须有 `\n\n`
- **新逻辑**：接受 `</thinking>` 后有任何空白字符（`\n`、`\n\n`、空格等）
- **保持**：跳过被引用字符包裹的标签（反引号、引号等）

```rust
// 原逻辑
if after_content.starts_with("\n\n") {
    return Some(absolute_pos);
}

// 新逻辑
let first_char = after_content.chars().next().unwrap();
if first_char.is_whitespace() {
    return Some(absolute_pos);
}
```

### 2. 统一缓冲区保留逻辑

**新增辅助函数 `calculate_safe_buffer_split`**：

```rust
fn calculate_safe_buffer_split(buffer: &str, reserve_len: usize) -> usize {
    let target_len = buffer.len().saturating_sub(reserve_len);
    find_char_boundary(buffer, target_len)
}
```

**替换所有手动计算的地方**：

```rust
// 原代码
let target_len = self.thinking_buffer.len().saturating_sub("<thinking>".len());
let safe_len = find_char_boundary(&self.thinking_buffer, target_len);

// 新代码
let safe_len = calculate_safe_buffer_split(&self.thinking_buffer, "<thinking>".len());
```

### 3. 改进 `process_content_with_thinking` 逻辑

**在查找结束标签时添加边界检测回退**：

```rust
if let Some(end_pos) = find_real_thinking_end_tag(&self.thinking_buffer) {
    // 找到标准结束标签，处理...
} else {
    // 尝试边界检测
    if let Some(end_pos) = find_real_thinking_end_tag_at_buffer_end(&self.thinking_buffer) {
        // 找到边界结束标签，处理...
    } else {
        // 没有找到结束标签，保留缓冲区等待更多内容
    }
}
```

**改进标签后空白字符的剥离**：

```rust
// 原代码：硬编码 "</thinking>\n\n"
self.thinking_buffer = self.thinking_buffer[end_pos + "</thinking>\n\n".len()..].to_string();

// 新代码：动态跳过所有空白字符
let after_tag_pos = end_pos + "</thinking>".len();
let remaining_start = self.thinking_buffer[after_tag_pos..]
    .find(|c: char| !c.is_whitespace())
    .map(|pos| after_tag_pos + pos)
    .unwrap_or(self.thinking_buffer.len());
self.thinking_buffer = self.thinking_buffer[remaining_start..].to_string();
```

### 4. 更新测试用例

更新 `test_find_real_thinking_end_tag_basic` 测试，适应新的逻辑：

```rust
// 新增测试用例
assert_eq!(find_real_thinking_end_tag("</thinking>\n"), Some(0));
assert_eq!(find_real_thinking_end_tag("</thinking> "), Some(0));

// 更新预期结果
assert_eq!(find_real_thinking_end_tag("</thinking>"), None); // 缓冲区为空，等待更多内容
```

## 修改文件清单

- `src/anthropic/stream.rs`
  - `find_real_thinking_end_tag` (68-106 行)：放宽结束条件
  - 新增 `calculate_safe_buffer_split` (约 120 行)：统一缓冲区处理
  - `process_content_with_thinking` (700-800 行)：添加边界检测回退，改进空白字符剥离
  - `test_find_real_thinking_end_tag_basic` (1454-1470 行)：更新测试用例

## 预期效果

- ✅ 修复标签泄漏问题：`</thinking>` 后无论是 `\n`、`\n\n` 还是其他空白字符都能正确识别
- ✅ 统一缓冲区处理逻辑：减少重复代码，降低出错风险
- ✅ 保持现有架构不变：不引入 FSM 等复杂机制
- ✅ 代码改动约 60 行：符合最小改动原则

## 风险评估

**低风险**：
- 修改集中在标签检测逻辑，不影响核心流式处理
- 保持向后兼容，原有的 `\n\n` 情况仍然正常工作
- 有完整的测试覆盖

**需要注意**：
- 需要运行完整测试套件，确保没有破坏现有功能
- 建议在实际环境中测试各种边界情况

## 后续优化建议

如果未来需要更强大的功能，可以考虑：

1. 支持多种 thinking 标签（`<think>`, `<reasoning>`, `<thought>`）
2. 添加配置选项（处理模式、最大长度等）
3. 重构为完整的 FSM 解析器（参考 kiro-gateway）

但当前的修复方案已经足够解决现有问题。
