# ER Diyagramı (taslak)

M1 aşamasındaki **planlanan** veri modelidir. Proje tanımındaki 10 tabloyu (users, patients, clinicians, medications, medication_schedules, dose_logs, measurements, coach_messages, alerts, education_contents) ve bakım veren ilişkisi için bir bağlantı tablosunu içerir. Tekrar gönderimlere karşı `idempotency_keys` tablosu da plana eklenmiştir. Kesin şema M2'de Alembic migration'larıyla oluşturulacaktır; web push abonelikleri, hatırlatma kayıtları ve LLM maliyet kaydı gibi yardımcı tablolar ilgili teslimde eklenecektir.

![er-diyagrami](diyagramlar/er-diyagrami.png)

<details><summary>Mermaid kaynağı</summary>

```mermaid
erDiagram
    users ||--o| patients : "hasta profili"
    users ||--o| clinicians : "hekim profili"
    clinicians ||--o{ patients : "takip eder"
    patients ||--o{ patient_caregivers : ""
    users ||--o{ patient_caregivers : "bakım veren"
    patients ||--o{ medications : ""
    medications ||--o{ medication_schedules : ""
    medication_schedules ||--o{ dose_logs : ""
    patients ||--o{ dose_logs : ""
    patients ||--o{ measurements : ""
    patients ||--o{ coach_messages : ""
    patients ||--o{ alerts : ""
    users ||--o{ idempotency_keys : "tekrar gönderim koruması"

    users {
        int id PK
        string email UK
        string password_hash
        string full_name
        enum role "patient|clinician|caregiver"
        datetime created_at
    }
    clinicians {
        int id PK
        int user_id FK
        string title
        string institution
    }
    patients {
        int id PK
        int user_id FK
        int clinician_id FK
        int birth_year
        bool has_diabetes
        bool has_hypertension
        datetime consent_at
    }
    patient_caregivers {
        int patient_id PK,FK
        int caregiver_user_id PK,FK
    }
    medications {
        int id PK
        int patient_id FK
        string name
        string active_ingredient
        string dose_text "hekimin yazdığı haliyle"
        bool active
    }
    medication_schedules {
        int id PK
        int medication_id FK
        time time_of_day
        string days_of_week "1234567"
    }
    dose_logs {
        int id PK
        int patient_id FK
        int schedule_id FK
        date scheduled_date
        enum status "taken|skipped"
        string reason
        datetime logged_at
    }
    measurements {
        int id PK
        int patient_id FK
        enum type "glucose_fasting|glucose_postprandial|blood_pressure"
        float value
        float value2 "diyastolik"
        string unit
        datetime measured_at
    }
    coach_messages {
        int id PK
        int patient_id FK
        enum sender "patient|coach"
        text content
        enum risk_level "low|medium|high|emergency"
        bool escalate_to_clinician
        json sources
        string guardrail_action
        datetime created_at
    }
    alerts {
        int id PK
        int patient_id FK
        enum severity "info|warning|critical"
        string kind
        text message
        datetime created_at
        datetime acknowledged_at
    }
    idempotency_keys {
        int id PK
        int user_id FK
        string key "istemcinin ürettiği UUID"
        string request_hash "aynı anahtar + farklı gövde → 422"
        int response_status
        json response_body
        datetime created_at "24 saat sonra temizlenir"
    }
    education_contents {
        int id PK
        string title
        text body
        string source_name
        string source_url
        string tags
        string kind "curated|openfda"
        vector embedding "pgvector"
        datetime created_at
    }
```

</details>

## Notlar

- `dose_logs (schedule_id, scheduled_date)` tekildir: bir planlı doz için gün başına tek kayıt.
- `medications.dose_text` hekimin yazdığı metindir; sistem bu alanı yorumlamaz ve değiştirmez (doz önerisi kapsam dışı).
- `coach_messages` koçun structured output alanlarını (`risk_level`, `escalate_to_clinician`, `sources`) ve guardrail kararını saklar; hekim yalnızca risk bayraklı mesajları görür.
- `education_contents` RAG korpusudur: `curated` kayıtlar elle seçilmiş Türkçe hasta eğitim metinleri, `openfda` kayıtları ilaç etiketlerinden üretilen özetlerdir.
- `idempotency_keys (user_id, key)` tekildir. Yazma isteği ile anahtar aynı transaction'da kaydedilir; aynı anahtar tekrar gelirse kayıtlı yanıt aynen döner (bkz. [mimari §3](05-mimari.md#3-bir-isteğin-backend-katmanlarından-geçişi)). M2'de eklenir.
- Hasta silindiğinde kişisel tüm kayıtları (cascade) silinir.
