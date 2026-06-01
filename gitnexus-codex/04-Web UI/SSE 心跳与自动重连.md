---
type: implementation
status: codex-generated
tags:
  - gitnexus
  - sse
  - web-ui
---

# SSE 心跳与自动重连

Web UI 通过 `/api/heartbeat` 维护浏览器和后端之间的连接状态。后端定期发送 SSE 注释，前端监听 open/error，断线时展示重连状态。

## 流程图

![[gitnexus-heartbeat-reconnect-sequence.png]]

## 关键点

- 后端每隔固定时间发送心跳，避免连接被静默关闭。
- 前端用 EventSource 监听连接状态。
- 断线后展示黄色提示，并使用指数退避重连。
- 这个机制提高 Web UI 对本地服务重启、网络抖动的容忍度。

## 相关笔记

- [[Web UI 实现原理]]
