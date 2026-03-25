JSON（JavaScript Object Notation）是一种轻量级的数据交换格式，核心特点是**简洁、易读、跨语言兼容**，常用于前后端数据传输或配置文件存储。

### 1. 核心数据类型

JSON 仅支持 6 种基础数据类型，语法规则严格，不允许多余逗号或注释。

- **字符串**：必须用双引号包裹，例如 `"name": "Alice"`。
- **数字**：支持整数、浮点数，例如 `"age": 25`、`"score": 98.5`。
- **布尔值**：仅为 `true` 或 `false`，例如 `"isStudent": true`。
- **数组**：用方括号 `[]` 包裹，元素用逗号分隔，元素类型可不同，例如 `"hobbies": ["reading", "running", 3]`。
- **对象**：用大括号 `{}` 包裹，由 “键值对” 组成，键必须是双引号字符串，例如 `"user": {"id": 1, "name": "Bob"}`。
- **null**：表示空值，例如 `"address": null`。

---

### 2. 标准格式示例

以下是一个包含多种类型的完整 JSON 示例，可直接用于数据传输：

json

```json
{
  "name": "Li Ming",
  "age": 22,
  "isGraduated": false,
  "major": "Computer Science",
  "skills": ["HTML", "CSS", "JavaScript"],
  "contact": {
    "phone": "123456789",
    "email": "liming@example.com"
  },
  "awards": null
}
```

---

### 3. 关键语法规则

- 键必须用**双引号**包裹，不能用单引号或不加引号（如 `name: "Li Ming"` 是错误的）。
- 字符串值必须用**双引号**包裹（如 `'Li Ming'` 是错误的）。
- 数组和对象的最后一个元素后，**不能加逗号**（如 `["a", "b",]` 是错误的）。
- 不支持注释（`// 这是注释` 会导致 JSON 解析失败）。

---

### 4. 常见用途

- 前后端数据交互：前端通过 JSON 向后端发送请求参数，后端通过 JSON 返回数据。
- 配置文件：很多程序（如 Node.js 项目）用 JSON 作为配置文件（如 `package.json`）。
- 数据存储：小型数据可直接用 JSON 文件存储，无需数据库。