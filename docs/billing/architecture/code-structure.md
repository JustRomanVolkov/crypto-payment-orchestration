# Структура кода Billing

[Billing](../README.md) → [Архитектура](README.md) → Структура кода

Это дерево основных файлов для [модульного монолита с общей БД](../roadmap/modular-monolith-shared-db.md). Оно показывает, куда поместить доменные правила, сценарии, адаптеры и точки входа. В Billing сейчас один агрегат Payment, поэтому его файлы находятся прямо в `domain/`; общие базовые классы ему не требуются. Фоновые задачи входят в исходное решение; другое размещение приложения и БД описано в [развитии Billing](../roadmap/README.md).

```text
billing/
├── domain/
│   ├── aggregate.py                    # PaymentAggregate
│   ├── entities.py                     # PaymentAttemptEntity
│   ├── value_objects.py                # Деньги, актив, снимок маршрута
│   ├── enums.py                        # Состояния Payment и Attempt
│   └── policies/
│       ├── cascade_policy.py           # Условия перехода к следующему PSP
│       ├── fixed_priority_routing_policy.py  # FixedPriorityRoutingPolicy
│       └── routing_policy_protocol.py  # RoutingPolicyProtocol
├── application/
│   ├── use_cases/
│   │   ├── create_payment_use_case.py
│   │   ├── get_payment_status_use_case.py
│   │   ├── start_payment_attempt_use_case.py
│   │   ├── advance_payment_cascade_use_case.py
│   │   ├── handle_payment_webhook_use_case.py
│   │   ├── apply_payment_observation_use_case.py
│   │   └── reconcile_payment_use_case.py
│   └── interfaces/
│       ├── payment_repository_protocol.py
│       ├── payment_unit_of_work_protocol.py
│       ├── observation_inbox_protocol.py
│       ├── payment_provider_protocol.py
│       └── product_protocols.py
├── infra/
│   ├── persistence/
│   │   ├── models.py
│   │   ├── sqlalchemy_payment_repository_adapter.py
│   │   └── sqlalchemy_payment_unit_of_work_adapter.py
│   ├── payment_providers/
│   │   ├── nowpayments_provider_adapter.py
│   │   └── nowpayments_webhook_verifier_adapter.py
│   ├── product/
│   │   └── in_process_product_adapter.py
│   └── task_queue/
│       └── taskiq_broker.py
└── presentation/
    ├── http/
    │   ├── checkout_router.py
    │   └── payment_webhook_router.py
    └── tasks/
        ├── payment_tasks.py
        └── payment_recovery_scanner.py
```

## Границы и обязанности

| Место | Что в нём находится |
|---|---|
| `domain/` | `PaymentAggregate` защищает [инварианты Payment](domain/payment.md), `PaymentAttemptEntity` хранит состояние попытки; деньги, актив и маршрут остаются отдельными значениями. Роль корня агрегата определяется его правилами и использованием, а не наследованием от общего `AggregateRoot`. |
| `domain/policies/` | `routing_policy_protocol.py` объявляет `RoutingPolicyProtocol`; `fixed_priority_routing_policy.py` содержит его исходную реализацию для [маршрута](routing/policy.md). `CascadePolicy` выражает [условие безопасного продолжения](routing/cascade.md); окончательное разрешение новой Attempt остаётся у агрегата. |
| `application/use_cases/` | Команды HTTP и фоновых исполнителей. `HandlePaymentWebhookUseCase` только проверяет и сохраняет входящее наблюдение; `ApplyPaymentObservationUseCase` позднее проверяет деньги и фиксирует решение. `StartPaymentAttemptUseCase` исполняет одну попытку, а `AdvancePaymentCascadeUseCase` проверяет право перейти после доказанного no-create и назначает следующую работу. [Ответственность всех сценариев](contracts/application.md). |
| `application/interfaces/` | Репозиторий, UoW и inbox наблюдений; `payment_provider_protocol.py` объявляет `PaymentProviderProtocol`, `ProviderWebhookVerifierProtocol` и необязательный `PaymentReferenceLookupProtocol`. `product_protocols.py` объявляет интерфейсы снимка покупки и предоставления её результата. |
| `infra/` | `models.py` содержит ORM-модели Billing, включая observations и audit. SQLAlchemy-адаптеры используют одну session в общей UoW. `nowpayments_provider_adapter.py` реализует вызовы по `PaymentProviderProtocol`; `nowpayments_webhook_verifier_adapter.py` реализует `ProviderWebhookVerifierProtocol`. Поддержка `PaymentReferenceLookupProtocol` добавляется только после проверки возможностей PSP. Адаптер продукта вызывает операцию модуля продукта, который сам владеет записью результата покупки. |
| `presentation/` | Входящие адаптеры HTTP и Taskiq передают команду соответствующему use case. Scanner находит сохранённую работу в PostgreSQL и передаёт её исполнителям; платёжные правила остаются в domain/application. Конфигурация `RedisStreamBroker` передаёт ID работы, а сама обязанность хранится в БД. |

Проверка PSP, ожидание его ответа и работа с внешним API выполняются вне SQL-транзакции. Для совместного commit результат покупки сохраняется операцией модуля продукта через общую UoW; условия описаны в [оплате и предоставлении покупки](flows/payment-to-product.md). В составе платформы зависимости связывает её точка сборки; отдельный файл `composition.py` внутри Billing для этого не обязателен.

Дерево относится к рекомендованному запуску. При раздельных транзакциях добавляются outbox и relay, при отдельном приложении — сетевые адаптеры; эти изменения показаны в [вариантах архитектуры](../roadmap/README.md). [Таблицы и ограничения](persistence.md) и [восстановление фоновой работы](processing.md) сохраняют свои правила при любом размещении.
