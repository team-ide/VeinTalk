# OpenAPI 接入说明

> 本文面向开发者，介绍系统开放能力的现状与规划。

> ⚠️ 当前状态：**OpenAPI 暂无**（规划中）。本文说明现状与可用的替代接入方式。

## 一、现状

系统暂未提供对外的 OpenAPI（appId / secret 鉴权的开放接口）。平台服务（server-platform）已预留相关服务定义，OpenAPI 开放后本文档将更新接入流程、签名规则与接口清单。

## 二、当前可用的接入方式

### 1. 客户端协议（不承诺对外兼容）

客户端（移动端 / Web）统一经接入网关访问：

- HTTP：`POST /vein-gateway/<path>`（JSON）
- 实时通道：WebSocket `/vein-gateway/access`（protobuf 二进制帧）

协议由 `proto` 定义，随客户端版本演进，**当前不对外承诺接口稳定性**，不建议第三方直接对接。

### 2. 移动端内置浏览器 JS-SDK（H5 应用接入）

移动端内置浏览器为 H5 应用提供 JS-SDK，可调用以下原生能力（含可信域名校验）：

- `getNetworkType` / `getSystemInfo` / `getAppInfo` — 环境信息
- `getUserInfo` / `getUserToken` / `getServerInfo` — 用户与服务器信息
- `copyText` / `showToast` / `setPageTitle` — 基础交互
- `closeWindow` / `openLink` — 窗口与链接
- `chooseImage`（1-9 张）/ `previewImage` — 图片能力
- `scanQRCode` — 扫码
- `getLocation` — 定位（GCJ-02 → WGS-84）

适合将已有 H5 应用嵌入移动端使用，是当前唯一对外开放的能力。

## 三、规划

- OpenAPI 开放平台：应用申请、appId / secret、签名规则、错误码体系
- Go SDK（客户端与服务端）
- 接口文档（按 proto 自动生成）

进度以 [更新日志](../05-版本发布/01-CHANGELOG.md) 与 [更新路线图](../05-版本发布/02-更新路线图.md) 为准。
