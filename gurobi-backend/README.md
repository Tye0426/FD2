# Gurobi SAN-A* 后端

GitHub Pages只能托管前端，不能运行Python或Gurobi。本目录需部署到一台具有有效Gurobi许可证的Python服务器，前端再填写其 `/solve` 地址。

## 本地启动

```bash
python -m venv .venv
source .venv/bin/activate   # Windows使用 .venv\Scripts\activate
pip install -r requirements.txt
export ALLOWED_ORIGINS=https://Tye0426.github.io
uvicorn app:app --host 0.0.0.0 --port 8000
```

Windows PowerShell设置来源：

```powershell
$env:ALLOWED_ORIGINS="https://Tye0426.github.io"
uvicorn app:app --host 0.0.0.0 --port 8000
```

健康检查：`GET /health`。求解接口：`POST /solve`。修正版健康检查会返回 `backendVersion: 1.1.0`、`poolSearchMode: 1` 和单个SAN模型的最长运行时间，可据此确认服务器已经更新。部署后还可访问 `GET /self-test`：它会实际建立并求解一个小型非平凡案例，同时检查Gurobi许可证、SAN候选生成和外层方案搜索；返回的 `ok` 应为 `true`。

## SAN子模型时间分配

后端不会再把全部剩余搜索时间交给第一个Gurobi SAN子模型。每个源股道模型使用公平短时预算，默认单个模型最多1秒，并优先收集可行池解，把剩余时间留给外层状态树。可通过环境变量调整：

```powershell
$env:GUROBI_SAN_MODEL_MAX_SECONDS="1.0"
$env:GUROBI_THREADS="0"
```

`GUROBI_THREADS=0`表示由Gurobi自动选择线程数。通常不建议把单模型上限提高到5秒以上，否则某个困难SAN模型仍可能占用大量总搜索时间。

后端接收页面的 `searchStrategy`（`auto`、`sanr`、`astar`、`dfs`、`brfs`、`best`、`cbfs`）。独立的SAN-R仅运行单路径奖励基线；其余策略先尝试短时SAN-R，必要时再通过合法SAN动作构造可行解，随后以该解作为上界运行所选分支树搜索。返回 `seedMethod`、`status`、`termination` 和每一步作业记录；时间或解池截断时有方案只标记为可行，不宣称全局最优。该模型仍与论文的有环SAN及延迟约束存在差异，详见上层《算法说明.md》。

生产环境必须使用HTTPS，否则HTTPS的GitHub Pages页面会阻止调用HTTP接口。Gurobi许可证由后端部署者自行配置；不要把许可证文件或密钥提交到GitHub。
