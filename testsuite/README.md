# Building Energy Data Product Onboarding Suite

## 概述

本目录包含 DSSC Group D 设计的完整 GITB TDL Test Suite，用于验证建筑能耗数据产品入驻 Data Space 的合规性。

## 目录结构

```
testsuite/
├── README.md                                     ← 本文件
├── testSuite.xml                                 ← Test Suite 入口定义
├── tests/
│   ├── tc-shacl-valid-metadata.xml               ← TC-1: 正确元数据 SHACL 验证
│   ├── tc-shacl-invalid-metadata.xml             ← TC-2: 错误元数据 SHACL 检测
│   ├── tc-api-response-format.xml                ← TC-3: API 响应格式验证
│   └── tc-license-compliance.xml                 ← TC-4: 许可协议合规验证
└── artifacts/
    ├── building-energy-shapes.ttl                ← SHACL 约束规则文件（9 条）
    ├── data-product-valid.jsonld                 ← 合法元数据样本
    ├── data-product-invalid.jsonld               ← 非法元数据样本（3 个错误）
    └── openapi.yaml                              ← API 接口规范
```

## Test Case 总览

| # | Test Case ID | 类型 | 依赖 | 验证内容 |
|---|-------------|------|:---:|---------|
| 1 | `urn:dssc:tc:shacl-valid-metadata` | CONFORMANCE | — | 正确元数据应通过 SHACL 全部 9 条约束 |
| 2 | `urn:dssc:tc:shacl-invalid-metadata` | CONFORMANCE | TC-1 | 错误元数据应被检测出 3 个违规 |
| 3 | `urn:dssc:tc:api-response-format` | CONFORMANCE | — | API 响应结构与 OpenAPI 一致 |
| 4 | `urn:dssc:tc:license-compliance` | CONFORMANCE | TC-3 | license 在允许的白名单中 |

## 使用方法

### 方式 1: 独立文件验证（对应 TC-1、TC-2、TC-4）

直接在 SEMIC SHACL Validator 在线页面执行：
- 访问 https://www.itb.ec.europa.eu/shacl/any/upload
- Shapes Graph: 上传 `artifacts/building-energy-shapes.ttl`
- Data Graph: 上传 `artifacts/data-product-valid.jsonld` 或 `artifacts/data-product-invalid.jsonld`

### 方式 2: 完整 Test Suite 部署（进阶，需本地 ITB）

```bash
# 打包
cd testsuite
zip -r energy-onboarding-suite.zip testSuite.xml tests/ artifacts/

# 部署到 ITB
# 登录 ITB → Test Suite Management → Upload → 选择 ZIP
```

## 验证流程图

```
数据产品入驻 Data Space 的合规验证流水线:

  ┌─────────────────────┐
  │  TC-1: Valid        │  ← 先验证引擎和规则正确
  │  SHACL → sh:conforms│
  │       = true        │
  └──────────┬──────────┘
             │ prerequisite
             ▼
  ┌─────────────────────┐
  │  TC-2: Invalid      │  ← 再验证引擎能正确检测错误
  │  SHACL → sh:conforms│
  │       = false       │
  │  3 个 ValidationRes │
  └─────────────────────┘

  ┌─────────────────────┐
  │  TC-3: API Response │  ← 独立运行：验证 API 格式
  │  Format Validation  │
  │  HTTP 200 / JSON    │
  └──────────┬──────────┘
             │ prerequisite
             ▼
  ┌─────────────────────┐
  │  TC-4: License      │  ← 验证许可协议合规
  │  Compliance         │
  │  CC-BY-4.0 白名单   │
  └─────────────────────┘

  全部通过 → Conformance Statement: 该数据产品可以入驻 Data Space
```

## 对应的验证规范

此 Test Suite 验证以下规范的实现：

- **Building Energy Metadata Spec v1.0** (C 组定义的语义模型)
- **building-energy-shapes.ttl** (9 条 SHACL 约束规则)
- **openapi.yaml** (API 接口契约)

## 设计者

DSSC Group D — Conformance & Validation
DSSC Toolbox Research Project
2026-06-25
