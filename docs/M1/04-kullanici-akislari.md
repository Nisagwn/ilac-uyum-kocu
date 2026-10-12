# Kullanıcı Akışları

## A. Hasta — günlük akış

```mermaid
flowchart TD
    A[Web push: "Metformin zamanı"] --> B[Bugün ekranı]
    B --> C{Dozu aldı mı?}
    C -- Aldım --> D[dose_log: taken]
    C -- Almadım --> E[dose_log: skipped + isteğe bağlı neden]
    D --> F[Ölçüm gir: açlık kan şekeri]
    E --> F
    F --> G{Eşik dışı mı?}
    G -- Evet --> H[Kritik uyarı + hekime bayrak]
    G -- Hayır --> I[Grafik güncellenir]
    I --> J[Koç kartı: günün geri bildirimi]
```

## B. Hasta — koç sohbeti

```mermaid
flowchart TD
    A[Hasta mesaj yazar] --> B[Katman 1: girdi filtresi]
    B -- Acil belirti --> C[Sabit 112 yanıtı + kritik alert]
    B -- Doz talebi --> D[Sabit güvenli yanıt + hekime not önerisi]
    B -- Temiz --> E[LLM koç]
    E --> F[Tool: get_adherence 7/30 gün]
    E --> G[Tool: get_measurements]
    E --> H[Tool: search_education RAG]
    F & G & H --> I[Structured output]
    I --> J[Katman 3: çıktı doğrulayıcı]
    J -- Geçerli --> K[Yanıt + kaynaklar]
    J -- İhlal --> L[Güvenli yedek yanıt]
    K --> M{escalate_to_clinician?}
    M -- Evet --> N[Hekime alert]
```

## C. Hekim — haftalık inceleme

```mermaid
flowchart TD
    A[Giriş] --> B[Hasta listesi: uyum skoruna göre sıralı]
    B --> C[Kırmızı bayraklı hastalar üstte]
    C --> D[Hasta detayı: 30 gün eğri + doz takvimi]
    D --> E[LLM haftalık özet]
    E --> F[Uyarıyı incele / kapat]
```

## D. Bakım veren

Hasta, ayarlardan e-posta ile bakım veren ekler → bakım veren hesabıyla giriş yapar → bağlı hastanın bugün ekranını, grafiklerini ve uyarılarını **salt okunur** görür.

## E. Acil belirti akışı

```mermaid
flowchart TD
    A["Hasta: göğsümde baskı var"] --> B{"Katman 1: acil belirti mi?"}
    B -- "evet" --> C["LLM ÇAĞRILMAZ"]
    C --> D["Kırmızı acil kartı + 112'yi ara düğmesi"]
    C --> E["Kritik alert: hekim paneline düşer"]
    D --> F["Hasta 112'yi arar / acil servise gider"]
    E --> G["Hekim hastayla iletişime geçer"]
    G --> H["Uyarıyı okundu işaretler"]
    B -- "hayır" --> I["Normal koç akışı"]
```

## F. Hekim — triyaj akışı

```mermaid
flowchart TD
    A["Hekim girişi"] --> B["Hasta listesi"]
    B --> C{"Açık kritik uyarı var mı?"}
    C -- "evet" --> D["Liste başında: önce bakılır"]
    C -- "hayır" --> E{"Uyum bandı"}
    E -- "düşük" --> F["Yukarıda: yakından izle"]
    E -- "orta / iyi" --> G["Rutin sıra"]
    D --> H["Hasta detayı: eğriler, doz takvimi, uyarılar"]
    F --> H
    G --> H
    H --> I["Uyarıyı okundu işaretle"]
```

## G. Bakım veren — akış

```mermaid
flowchart TD
    A["Hasta ayarlardan bakım veren ekler"] --> B["Bakım veren hesabıyla giriş"]
    B --> C["Bağlı hastanın bugün ekranı"]
    C --> D["Grafikler ve uyarılar"]
    D --> E{"Düzenleme denemesi?"}
    E -- "evet" --> F["Reddedilir: salt okunur"]
    E -- "hayır" --> C
```

