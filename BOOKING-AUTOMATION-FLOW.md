# Booking Automation Flow

## 1. Automation campaign là gì

```
campaigns_v2                          automation_campaigns
├─ venueId                            ├─ triggerName        reservation.created | booking.made | ...
├─ type      = 'automation'           ├─ conditionConfig    JSONB { whenToSend, conditions, nextDayAt }
├─ status    = 'active'               ├─ customTimingValue  2
├─ channel   = 'email'                ├─ customTimingUnit   hours | days | weeks | months
├─ content   { html, emailSubject }   ├─ customTimingDirection before | after
└─ automationConfigId ────────────────┘  automationPurpose   'transactional' (system-provisioned)
```

| Trigger (native) | Key | Default timing | Enabled |
|---|---|---|---|
| Booking created | `reservation.created` | immediate | yes |
| Booking modified | `reservation.modified` | immediate | yes |
| Booking cancelled | `reservation.cancelled` | immediate | yes |
| Confirmation request | `reservation.confirmation_due` | 24h before | yes |
| Reminder | `reservation.reminder_due` | 2h before | no |
| No-show | `reservation.no_show` | next day 09:00 | no |
| Post-visit | `reservation.completed` | 3h after | yes |
| Waitlist added / table available | `reservation.waitlist.*` | immediate | no |

Recipes: `src/constants/reservation-automation-recipes.constant.ts`
Enums: `src/constants/enums/campaign.enum.ts`

## 2. Luồng native nollie Bookings

```mermaid
flowchart TD
    A["1. Create booking<br/>staff / widget / agent"] --> B["2. reservation.service.ts<br/>emitReservationCreatedSqsEvent"]

    B --> B1["3a. Send SQS<br/>RESERVATION_CREATED<br/>{ venueId, reservationId }"]
    B --> B2["3b. Schedule delayed jobs<br/>confirmation request, reminder"]
    B2 --> J[("reservation_automation_jobs<br/>status=scheduled, dueAt")]
    J --> C["4. Cron every minute<br/>reservation-automation.scheduler.ts<br/>dueAt <= now → status=enqueued"]
    C --> C1["5. Send SQS<br/>CONFIRMATION_DUE / REMINDER_DUE<br/>NO_SHOW / COMPLETED"]

    B1 --> Q[("SQS<br/>Notifications queue")]
    C1 --> Q

    Q --> L["6. sqs-listener.consumer.ts<br/>worker only"]
    L --> N["7. reservation-notification.service.ts<br/>dispatchNativeBookingRecipes"]
    N --> F{"8. feature flag<br/>automation_native_bookings"}
    F -->|off| X[stop]
    F -->|on| G["9. Reload reservation + venue + customer<br/>re-check gates"]
    G --> M["10. campaigns_v2 ACTIVE + AUTOMATION<br/>JOIN automation_campaigns WHERE triggerName"]
    M --> R["11. for each recipe → sendRecipeOnce"]
    R --> I[("12. reservation_automation_send_log<br/>UNIQUE campaign_id + reservation_id + event_key")]
    I -->|UniqueConstraintError| X
    I -->|claimed| T["13. Resolve merge tags<br/>render html + subject"]
    T --> S["14. SendGrid<br/>booking client"]
    S -->|fail| D["15. delete send_log row → retry on redelivery"]
```

Thứ tự:

| # | Bước | Ghi chú |
|---|---|---|
| 1-2 | API tạo booking, gọi `emitReservationCreatedSqsEvent` | fire-and-forget, không block response |
| 3a | Bắn SQS `RESERVATION_CREATED` | trigger gửi ngay: `reservation.created` |
| 3b | Ghi `reservation_automation_jobs` | trigger có delay: `confirmation_due`, `reminder_due`; `no_show` và `completed` ghi khi status booking đổi |
| 4-5 | Cron mỗi phút quét job đến hạn, bắn SQS | chỉ với trigger có delay |
| 6-7 | Worker nhận SQS, route theo `type` sang service | |
| 8-9 | Check flag + gate lại tại thời điểm gửi | |
| 10-11 | Match campaign theo `venueId` + `triggerName` | |
| 12 | Claim idempotency | trùng → bỏ qua |
| 13-14 | Render + gửi SendGrid | |
| 15 | Fail → release claim | SQS redeliver sẽ retry |
