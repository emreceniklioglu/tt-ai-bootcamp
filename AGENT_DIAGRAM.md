# Bundlekom Agent — Akis Diyagrami

```mermaid
flowchart TD
    U(["Kullanici mesaji"]) --> IG{"Girdi guardraillari<br/>bos / uzunluk / injection"}
    IG -- "ihlal" --> RED1["Kibar red mesaji"] --> OUT(["Kullaniciya yanit"])
    IG -- "ok" --> TOPIC{"Acik konu var mi?"}

    TOPIC -- "evet" --> RES{"Cozum durumu<br/>(LLM siniflandirma)"}
    RES -- "RESOLVED" --> SAVE["Sohbeti Soru + Cozum olarak<br/>reference_material.json'a yaz<br/>(DOC001001+) ve Chroma'ya<br/>aninda ingest et"]
    SAVE --> ASK["Baska konuda yardimci<br/>olabilir miyim?"] --> OUT
    RES -- "NOTRESOLVED" --> ATT{"Deneme sayisi < 2 ?"}
    ATT -- "evet" --> RERAG["Onceki dokumanlar haric (exclude_ids)<br/>tekrar RAG + alternatif cozum iste"] --> OG
    ATT -- "hayir" --> CHURN["TOOL: churn skoru<br/>(LightGBM lookup CSV,<br/>yoksa sezgisel yedek)"]
    CHURN --> URG["Aciliyet belirle<br/>skor >= 0.50: yuksek / 24 saat<br/>skor >= 0.20: orta / 48 saat<br/>alti: standart / 72 saat"]
    URG --> CB["Temsilci arama talebi olustur<br/>callback_requests.json (CB-xxxxx)<br/>'durumunuz iletildi' mesaji"] --> OUT
    RES -- "NEWTOPIC" --> GATE

    TOPIC -- "hayir" --> GATE{"LLM konu kapisi<br/>(Bundlekom ile ilgili mi?)"}
    GATE -- "GREETING" --> HELLO["Karsilama + yetenek listesi"] --> OUT
    GATE -- "OFFTOPIC" --> RED2["'Uzgunum, bu konuda bilgim yok.<br/>Su konularda yardimci olabilirim...'"] --> OUT
    GATE -- "ONTOPIC" --> INT{"Intent tespiti<br/>(few-shot ICL)"}

    INT -- "BILLING" --> BILL["SQLite: son fatura + gecmis<br/>tarihler 03-2024 formatinda<br/>konunun ILK mesajinda ham string<br/>dogrudan kullaniciya basilir"]
    INT -- "DATA" --> DATA["SQLite: guncel + gecmis<br/>data kullanimi (GB)"]
    INT -- "OTHER" --> CAT
    BILL --> CAT["Kategori tespiti<br/>(10 kategori, dokumanlardan<br/>gercek orneklerle ICL)"]
    DATA --> CAT
    CAT --> RAGQ["Chroma sorgusu: top-10<br/>kategori metadata filtresi<br/>(belirsizse filtresiz)"]
    RAGQ --> RR["Cross-encoder rerank<br/>10 dokuman -> en iyi 3"]
    RR --> GEN["Gemini cevap uretimi<br/>- musteri verisini AYNEN aktar<br/>- dokuman mantiksizsa yorumsuz:<br/>'Dokumanda su bilgiyi buldum'<br/>- sonda 'cozuldu mu?' diye sor"]
    GEN --> OG{"Cikti guardraillari<br/>TL/GB sayi tutarliligi<br/>prompt sizintisi / baska musteri PII"}
    OG -- "ok" --> OUT
    OG -- "ihlal" --> FB["Guvenli fallback:<br/>ham veri + yorumsuz dokuman"] --> OUT
```

## Okuma notlari

- **Acik konu**: Asistan bir cozum onerdikten sonra konu "acik" kalir; bir
  sonraki mesaj once cozum geri bildirimi mi (RESOLVED / NOTRESOLVED) yoksa
  yeni bir soru mu (NEWTOPIC) diye siniflandirilir.
- **LLM cagrilari** (mesaj basina 3-4 adet, hepsi flash): konu kapisi,
  intent/cozum-durumu, kategori, cevap uretimi.
- **State**: sohbet gecmisi (`history`), acik konu (`topic`: intent, kategori,
  deneme sayisi, kullanilan dokuman id'leri, acilis mesaji), oturum sonunda
  `data/sessions/` altina JSON olarak yazilir.
- **Veri katmani**: `data/bundlekom.db` (SQLite, 4.3M harcama satiri) ve
  `data/chroma/` (1000+ dokuman; cozulen sohbetlerle buyur).
