# Validator SOAP 接入冒烟测试套件

这个目录提供一个最小 GITB TDL Test Suite，用于验证以下调用链是否成立：

```text
ITB Test Case
  -> Docker 内部 SOAP WSDL
  -> 独立 SHACL Validator（energy/v1）
  -> building-energy-shapes_D.ttl
  -> GITB 验证报告
  -> ITB 测试会话
```

## 打包

在本目录执行：

```powershell
Compress-Archive `
  -Path ".\testSuite.xml", ".\testCases" `
  -DestinationPath ".\energy-validator-soap-smoke.zip" `
  -Force
```

ZIP 根目录必须直接包含 `testSuite.xml` 和 `testCases` 文件夹。

## 执行

将 ZIP 上传到 ITB 后运行 `Upload and validate Building Energy metadata`：

- 上传 `..\testsuite\artifacts\data-product-valid.jsonld`，预期通过；
- 上传 `..\testsuite\artifacts\data-product-invalid.jsonld`，预期失败并显示 SHACL 报告。

这个套件只验证 SOAP 接入，不替代完整的 onboarding Test Suite。
