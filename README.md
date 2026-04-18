
# Emotion Support Backend

一个可运行的最小后端骨架，面向 **OpenAvatarChat (OAC) 前端 + vLLM 推理服务** 的第二阶段开发。

## 功能
- FastAPI API 网关
- PostgreSQL 数据库存储策略、对话日志、记忆快照、反馈、候选策略
- 策略从数据库读取，不写死在代码里
- 根据用户输入做简单风险识别、场景识别、情绪识别、策略选择
- 使用 vLLM 的 OpenAI-compatible Chat API 生成回复
- 记录反馈并动态更新策略分数
- 导出训练数据，供 LLaMA-Factory 做 SFT / DPO

## 目录
- `app/` 线上服务
- `trainer/` 离线数据导出与训练配置
- `scripts/` 初始化数据库、种子策略、回填分数
- `environment.yml` Conda 环境定义
- `docker-compose.yml` 可选：仅启动 PostgreSQL / Redis

## 推荐运行方式（Anaconda）
### 1) 创建 Conda 环境
```bash
conda env create -f environment.yml
conda activate emotion-support-backend
```

### 2) 配置环境变量
复制 `.env.example` 为 `.env`，按需修改：
```bash
cp .env.example .env
```

### 3) 启动 PostgreSQL（可选 Docker）
```bash
docker compose up -d postgres redis
```

### 4) 初始化数据库与种子策略
```bash
python scripts/init_db.py
python scripts/seed_strategies.py
```

### 5) 启动后端
```bash
python run_api.py
```

默认监听：`http://127.0.0.1:8010`

## 接口
### 健康检查
```bash
curl http://127.0.0.1:8010/health
```

### 聊天接口
```bash
curl -X POST http://127.0.0.1:8010/chat/respond \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": "u001",
    "session_id": "s001",
    "text": "最近论文和答辩压力太大了，晚上总是睡不着",
    "top_k_history": 6
  }'
```

### 反馈接口
```bash
curl -X POST http://127.0.0.1:8010/feedback \
  -H "Content-Type: application/json" \
  -d '{
    "conversation_id": 1,
    "score": 1,
    "is_good_strategy": true,
    "is_good_response": true
  }'
```

## 对接 OAC
把 OAC 当前的 LLM 请求地址从 vLLM 直连，改为你的后端 `/chat/respond` 即可。  
你的后端内部再调用 vLLM。

## vLLM
vLLM 可作为 OpenAI-compatible server 启动，例如：
```bash
vllm serve Qwen/Qwen2.5-3B-Instruct --api-key EMPTY --host 0.0.0.0 --port 8000
```

## 训练
1. 用 `trainer/build_dataset.py` 导出 SFT 样本
2. 用 `trainer/export_feedback_pairs.py` 导出偏好数据
3. 结合 LLaMA-Factory 的 YAML 配置训练
4. 合并 LoRA 后，更新 vLLM 所加载模型
