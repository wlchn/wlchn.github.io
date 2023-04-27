---
title: "Kubernetes等待部署完成-kubectl-wait-rollout"
date: 2019-03-18T00:00:00+08:00
---

使用 kubectl apply 或者 create 命令创建/更新部署后，其 pod 需要时间完成创建/更新。
如果在 CI 中不等待所有 pod 更新完成，下一步流程很有可能使用更新前的环境。

## 正确方法 kubectl rollout status

根据实际情况`kubectl rollout status`命令，能够正确的获取 rollout status。

```shell
ATTEMPTS=0
ROLLOUT_STATUS_CMD="kubectl rollout status deployment/myapp -n namespace"
until $ROLLOUT_STATUS_CMD || [ $ATTEMPTS -eq 60 ]; do
  $ROLLOUT_STATUS_CMD
  ATTEMPTS=$((attempts + 1))
  sleep 10
done
```

以上 shell 脚本经过轮询`kubectl rollout status`可以很好的解决实际问题。

## 错误方法 kubectl wait

注意`kubectl wait`命令不能适用于更新部署。

```shell
kubectl wait --for=condition=available --timeout=600s deployment/myapp -n namespace
```

该命令只能判断 deployment 是否 available，不能用来判断 rollout，即 available 状态的 deployment，很可能老的 pod 还在 terminating，新的 pod 还没创建好。

## references

- https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
- https://www.jeffgeerling.com/blog/2018/updating-kubernetes-deployment-and-waiting-it-roll-out-shell-script
