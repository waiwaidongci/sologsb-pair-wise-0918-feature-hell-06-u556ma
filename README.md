# 机械钟表擒纵调校API

纯后端零依赖Node服务，使用 `data/db.json` 持久化钟表档案、调校记录、复测记录和零件领用记录。

## 启动

```bash
PORT=3021 node server.js
```

## 主要接口

- `GET /health`
- `GET /clocks`
- `POST /clocks`
- `GET /clocks/not-qualified`
- `GET /clocks/:id/history`
- `POST /clocks/:id/adjustments`（可在 `partInstallations` 中登记零件安装结果）
- `POST /clocks/:id/retests`（存在未安装零件时返回409）
- `GET /clocks/:id/latest-retest`
- `GET /clocks/:id/part-claims?status=`
- `POST /clocks/:id/part-claims`
- `POST /clocks/:id/part-claims/:claimId/cancel`
- `GET /adjustments?clockId=`
- `GET /retests?clockId=&qualified=`
- `GET /part-claims?clockId=&status=&serialNumber=`

## 零件领用与安装闭环

1. **领用**：`POST /clocks/:id/part-claims`，按唯一序列号 `serialNumber` 领用零件。同一序列号已被领用（active）或已安装（installed）时返回 **409**，且不写入数据。
2. **登记安装**：领用后必须在调校中登记安装结果——`POST /clocks/:id/adjustments` 携带 `partInstallations: [{ "claimId": "...", "installed": true, "note": "..." }]`。已安装/已取消的领用重复登记返回 **409**。
3. **复测拦截**：存在未安装（active）领用时，`POST /clocks/:id/retests` 返回 **409**，不落库。
4. **取消领用**：`POST /clocks/:id/part-claims/:claimId/cancel`，仅未安装的领用可取消；取消后该序列号可再次领用，历史领用记录仍保留可查。
5. **列表状态**：`GET /clocks` 中每个钟表带 `currentParts`（当前领用/已安装零件）、`pendingInstallations`（待安装数量）和 `retestEligible`（是否可复测）。

## 闭环示例

```bash
curl http://127.0.0.1:3021/clocks/not-qualified

# 领用零件
curl -X POST http://127.0.0.1:3021/clocks/clock_demo/part-claims \
  -H 'Content-Type: application/json' \
  -d '{"serialNumber":"SN-BALANCE-001","note":"更换摆轮"}'

# 调校并登记安装结果
curl -X POST http://127.0.0.1:3021/clocks/clock_demo/adjustments \
  -H 'Content-Type: application/json' \
  -d '{"currentDailyRateSeconds":45,"direction":"慢针方向","amount":"快慢针向慢侧0.3格","partInstallations":[{"claimId":"<claimId>","installed":true}]}'

# 复测
curl -X POST http://127.0.0.1:3021/clocks/clock_demo/retests \
  -H 'Content-Type: application/json' \
  -d '{"dailyRateSeconds":12,"amplitude":252,"note":"复测进入目标范围"}'
```
