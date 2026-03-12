# Feishu Document Sync Skill

## 用途
安全、可靠地将本地 Markdown 文档同步到飞书云文档，避免常见的格式错乱和内容丢失问题。

## 使用场景
- PRD 文档同步
- Backlog 文档同步
- 竞品分析报告同步
- 任何需要本地→飞书同步的文档

## 核心原则

### 1. 验证优先
- **写入后必须验证**：使用 `read` 或 `list_blocks` 检查内容
- **检查章节完整性**：确保 Heading1 在开头，章节顺序正确
- **检查关键内容**：确认表格、代码块、列表等元素完整

### 2. 操作顺序
```
1. 本地准备 → 2. 飞书写入 → 3. 内容验证 → 4. 确认完成
```

### 3. 错误处理
- 内容为空/错乱 → 删除重建
- URL变更 → 及时通知相关人员
- 权限问题 → 检查 owner_open_id

## 标准流程

### 新建文档
```javascript
// 1. 创建文档
const doc = await feishu_doc.create({
  title: "文档标题",
  content: content,  // 完整内容
  owner_open_id: "用户ID"
});

// 2. 验证内容
const verify = await feishu_doc.read({
  doc_token: doc.document_id
});

// 3. 检查关键指标
if (verify.block_count < 10 || !verify.content.includes("Heading1")) {
  // 内容异常，删除重建
  await feishu_drive.delete({ file_token: doc.document_id });
  throw new Error("内容写入失败，需要重建");
}

// 4. 返回新URL
return doc.url;
```

### 更新文档
```javascript
// 1. 写入新内容
await feishu_doc.write({
  doc_token: token,
  content: content
});

// 2. 验证内容
const verify = await feishu_doc.read({ doc_token: token });

// 3. 检查章节顺序
const blocks = await feishu_doc.list_blocks({ doc_token: token });
const firstHeading = blocks.blocks.find(b => b.block_type === 3); // Heading1
if (!firstHeading || firstHeading.block_id !== blocks.blocks[1].block_id) {
  // 章节错乱，删除重建
  await feishu_drive.delete({ file_token: token });
  throw new Error("章节顺序错乱，需要重建");
}

// 4. 确认完成
return "同步成功";
```

## 常见错误

| 错误现象 | 原因 | 解决方案 |
|----------|------|----------|
| 文档只有标题 | `create`内容未写入 | 使用`write`补充 |
| 章节错乱 | `write`保留旧block顺序 | 删除重建 |
| 内容重复 | 新旧内容叠加 | 清空后写入或重建 |
| 表格丢失 | 格式解析错误 | 检查markdown语法 |

## 验证检查清单

- [ ] 文档标题正确
- [ ] Heading1 在文档开头
- [ ] 章节编号连续（1, 2, 3...）
- [ ] 关键表格完整
- [ ] 代码块保留
- [ ] 列表层级正确
- [ ] 链接可点击
- [ ] 总block数合理

## 工具函数

### 快速验证
```javascript
async function verifyDoc(token, expectedBlocks = 100) {
  const blocks = await feishu_doc.list_blocks({ doc_token: token });
  
  // 检查block数量
  if (blocks.blocks.length < expectedBlocks) {
    return { ok: false, error: "Block数量不足" };
  }
  
  // 检查第一个非page block是否为Heading1
  const firstContent = blocks.blocks.find(b => b.block_type !== 1);
  if (firstContent?.block_type !== 3) {
    return { ok: false, error: "章节顺序错乱" };
  }
  
  return { ok: true };
}
```

## 注意事项

1. **删除重建会改变URL** - 需要通知相关人员更新链接
2. **大文档分段写入** - 超过5000字符建议分段
3. **保留本地备份** - 飞书文档异常时可从本地恢复
4. **定期验证** - 重要文档建议定期抽查验证

## 版本历史

- v1.0 (2026-03-12): 初始版本，基于StreamGuard PRD同步经验总结
