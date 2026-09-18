# 机械钟表擒纵调校API

纯后端零依赖Node服务，使用 `data/db.json` 持久化钟表档案、调校记录和复测记录。

## 启动

```bash
PORT=3021 node server.js
```

## 主要接口

- `GET /health`
- `GET /clocks`（列表含 `currentPart` 当前零件、`canRetest` 可复测状态及 `retestBlockReason`）
- `POST /clocks`
- `GET /clocks/not-qualified`
- `GET /clocks/:id/history`（含 `partRequisitions` 领用历史与 `installations` 安装历史）
- `POST /clocks/:id/adjustments`
- `POST /clocks/:id/retests`（未登记成功安装的零件时返回 409）
- `GET /clocks/:id/latest-retest`
- `POST /clocks/:id/part-requisitions` 领用零件（序列号全局唯一，重复领用返回 409）
- `GET /clocks/:id/part-requisitions` 查本钟表全部领用（含已取消）
- `POST /clocks/:id/part-requisitions/:rid/installations` 在当前调校登记安装结果（`result: success|failed`）
- `POST /clocks/:id/part-requisitions/:rid/cancel` 取消领用（已安装返回 409）
- `GET /part-requisitions?clockId=&serialNumber=&status=` 全局领用查询（历史可追溯）
- `GET /adjustments?clockId=`
- `GET /retests?clockId=&qualified=`

## 零件领用与安装闭环规则

1. 零件按唯一序列号同时只能被一个钟表领用：序列号存在未取消的领用时，再次领用（无论领用中还是已安装）返回 `409`，且不写库。
2. 领用后必须在**当前调校**（最近一条调校记录）上登记安装结果；安装失败保持「领用中」，可重试安装或取消。
3. 复测前置条件：当前调校存在、至少一件成功安装、且没有「领用中」未处理的零件；否则 `POST /retests` 返回 `409` 不落库。
4. 取消领用仅释放序列号占用（状态置为 `canceled`），该序列号可被任意钟表再次领用；历史领用记录保留，可按序列号查询。
5. 钟表列表的 `currentPart` 优先指向当前调校已安装的零件，否则为最近领而未装的零件，均无则为 `null`。

## 闭环示例

```bash
# 1. 发现不合格钟表（未装零件时 canRetest=false）
curl http://127.0.0.1:3021/clocks/not-qualified

# 2. 新建一轮调校
curl -X POST http://127.0.0.1:3021/clocks/clock_demo/adjustments \
  -H 'Content-Type: application/json' \
  -d '{"currentDailyRateSeconds":31,"direction":"慢针方向","amount":"更换摆轮轴后微调快慢针"}'

# 3. 领用零件（序列号唯一）
curl -X POST http://127.0.0.1:3021/clocks/clock_demo/part-requisitions \
  -H 'Content-Type: application/json' \
  -d '{"serialNumber":"BAL-2026-0001","partName":"摆轮组件","specification":"18000vph"}'
# 重复领用同序列号 -> 409，不落库

# 4. 在当前调校登记安装（未登记成功安装直接复测 -> 409）
curl -X POST http://127.0.0.1:3021/clocks/clock_demo/part-requisitions/<requisitionId>/installations \
  -H 'Content-Type: application/json' \
  -d '{"result":"success","position":"摆轮夹板","technician":"王师傅"}'

# 5. 复测（canRetest=true 才允许）
curl -X POST http://127.0.0.1:3021/clocks/clock_demo/retests \
  -H 'Content-Type: application/json' \
  -d '{"dailyRateSeconds":12,"amplitude":252,"note":"复测进入目标范围"}'

# 领用后改主意可取消（已安装不可取消），取消后序列号可再次领用，历史仍可查
curl -X POST http://127.0.0.1:3021/clocks/clock_demo/part-requisitions/<requisitionId>/cancel \
  -H 'Content-Type: application/json' -d '{"reason":"型号不匹配"}'
curl 'http://127.0.0.1:3021/part-requisitions?serialNumber=BAL-2026-0001'
```
