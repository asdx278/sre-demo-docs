# PodInfo: ошибки и рост времени ответа

Алерт: `PodInfoHighLatency`  
Severity: `critical`  
Сервис: `podinfo`, namespace `podinfo`  
Impact: пользователи получают медленные ответы или ошибки

## Симптомы

- p95 времени ответа PodInfo выше 500 мс дольше минуты
- Пришло уведомление из Grafana OnCall (эскалация SRE Primary → SRE Secondary → Lead SRE)
- Возможен рост ответов 4xx/5xx

## Быстрые проверки (первые 5 минут)

1. Состояние подов — все 4 реплики должны быть `Running` и `Ready`, без рестартов:

```bash
    kubectl -n podinfo get pods -o wide
```

2. Последние события и изменения — был ли недавно релиз:

```bash
    kubectl -n podinfo get events --sort-by=.lastTimestamp | tail -20
    kubectl -n podinfo rollout history deployment/podinfo
```

3. Логи приложения:

```bash
    kubectl -n podinfo logs deployment/podinfo --tail=100
```

4. Нет ли постороннего пода, генерирующего нагрузку:

```bash
    kubectl -n podinfo get pods | grep -v '^podinfo-'
```

5. Дашборд / Grafana Explore — деградирует один под или все:

```
    histogram_quantile(0.95, sum by (pod, le) (rate(http_request_duration_seconds_bucket{namespace="podinfo"}[2m])))
```

## Восстановление (один безопасный шаг по ситуации)

- **Источник — посторонний под с нагрузкой** (например, `podinfo-latency-chaos`): удалить его

```bash
    kubectl -n podinfo delete pod podinfo-latency-chaos
```

- **Недавно был релиз:** откатить последнее изменение

```bash
    kubectl -n podinfo rollout undo deployment/podinfo
    kubectl -n podinfo rollout status deployment/podinfo
```

- **Релизов не было, деградируют поды:** перезапуск без простоя (rolling restart)

```bash
    kubectl -n podinfo rollout restart deployment/podinfo
    kubectl -n podinfo rollout status deployment/podinfo
```

## Проверка после восстановления

- 4/4 пода `Ready`
- p95 < 500 мс в течение 5 минут
- Алерт перешёл в `resolved` в Grafana OnCall

## Эскалация

Не удалось восстановить за 15 минут — эскалировать Lead SRE через OnCall (Escalate) и сообщить в канал инцидента.
