# 直接访问 valkey

Valkey 是 `ate-api-server` 用来跟踪 actor 和 worker 记录的状态存储。直接访问它有助于调试状态相关的问题。

> **警告：** 请勿在正在运行的集群上执行破坏性命令（`FLUSHALL`、`DEL` 等）。

要打开一个 `valkey-cli` 会话：

1. `kubectl exec -n=ate-system -it valkey-cluster-0 -- valkey-cli -h valkey-cluster-service -c --tls --cacert /etc/valkey-ca/ca.crt --cer
t /run/servicedns.podcert.ate.dev/credential-bundle.pem --key /run/servicedns.podcert.ate.dev/credential-bundle.pem`
