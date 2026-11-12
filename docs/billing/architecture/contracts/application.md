# Application: сценарии и интерфейсы

[Billing](../../README.md) → [Архитектура](../README.md) → Контракты

Use case принимает команду HTTP или фоновой задачи и координирует [агрегат Payment](../domain/payment.md) с внешними зависимостями. Файлы и имена приведены в [структуре кода](../code-structure.md); документы `flows/` показывают порядок действий между процессами, БД и PSP.

## Сценарии

| Use case | Ответственность | Где участвует |
|---|---|---|
| `CreatePaymentUseCase` | Проверить покупку и повтор команды; сохранить Payment, маршрут и следующую работу, без вызова PSP. | [Создание оплаты](../flows/checkout.md) |
| `GetPaymentStatusUseCase` | Прочитать сохранённый статус и реквизиты для владельца без вызова PSP. | [Клиентское чтение](../flows/checkout.md) |
| `StartPaymentAttemptUseCase` | Под блокировкой проверить текущий допуск, подготовить Attempt и зафиксировать `CREATING` до внешнего `POST`; сохранить результат или неизвестный исход. | [Создание счёта](../flows/checkout.md) |
| `AdvancePaymentCascadeUseCase` | После доказанного no-create проверить право продолжить по сохранённому маршруту и назначить следующую работу; неизвестный исход и существующий счёт блокируют переход. | [Безопасный каскад](../routing/cascade.md) |
| `HandlePaymentWebhookUseCase` | Проверить подлинность и сохранить исходное наблюдение до ответа `2xx`; денежное решение здесь не принимается. | [Приём webhook](../flows/settlement.md) |
| `ApplyPaymentObservationUseCase` | В фоне связать сохранённое наблюдение с Attempt, проверить деньги и одной транзакцией зафиксировать решение, отметку применения и grant либо outbox. Получает также наблюдения от create и GET. | [Подтверждение оплаты](../flows/settlement.md) |
| `ReconcilePaymentUseCase` | Выполнить разрешённый GET/lookup, сохранить новое наблюдение или назначить следующий срок; неопределённый create не превращает в безопасный отказ по тайм-ауту. | [Сверка](../flows/reconciliation.md) |

Подготовка новой Attempt и решение о продолжении всегда проходят через агрегат: разделение сценариев не позволяет двум workers начать два create. Вызов PSP не удерживает SQL-транзакцию; условия продолжения после отказа приведены в [каскаде](../routing/cascade.md). [Денежные исключения](../flows/exceptions.md) остаются правилами применения наблюдений и разбора, а не отдельной командой без определённого интерфейса оператора.

## Интерфейсы зависимостей

`*Protocol` — Python-интерфейс на основе `typing.Protocol`. Application объявляет требуемую операцию, `infra/` предоставляет реализацию. Несколько связанных интерфейсов могут находиться в одном файле; это не сливает их права и транзакционные границы.

| Файл `application/interfaces/` | Интерфейсы и зависимость |
|---|---|
| `payment_repository_protocol.py` | `PaymentRepositoryProtocol` — загрузка и сохранение Payment; репозиторий не делает commit. |
| `payment_unit_of_work_protocol.py` | `PaymentUnitOfWorkProtocol` — одна локальная транзакция для Billing и, когда возможно, операции продукта. |
| `observation_inbox_protocol.py` | `ObservationInboxProtocol` — долговременное сохранение внешних свидетельств и отметки их применения. |
| `payment_provider_protocol.py` | `PaymentProviderProtocol` — запросы PSP (`infra/payment_providers/nowpayments_provider_adapter.py`); `ProviderWebhookVerifierProtocol` — проверка webhook (`infra/payment_providers/nowpayments_webhook_verifier_adapter.py`); необязательный `PaymentReferenceLookupProtocol` — поиск по reference только при доказанной возможности провайдера. [Контракт PSP](provider-adapter.md). |
| `product_protocols.py` | `PurchaseSnapshotProtocol` получает авторизованные условия покупки; `FulfillmentProtocol` сохраняет результат через операцию модуля продукта при общей транзакции. При раздельных транзакциях Billing сохраняет outbox вместо локального grant. |

Интерфейсы и DTO не передают HTTP-клиент или ORM-сессию в домен. Правила общей транзакции и раздельной доставки описаны в [оплате и предоставлении покупки](../flows/payment-to-product.md).
