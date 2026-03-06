# Call-Center QA Pipeline

Call-center keyfiyyətə nəzarət sistemi. Audio transkriptləri analiz edərək operatorun 5 meyar üzrə qiymətləndirilməsini həyata keçirir.

## Arxitektura
```
Input JSON → Validator → PII Detector → Rule Engine → LLM (Groq) → Score Aggregator → Output JSON
```

### Hibrid yanaşma (Rule-based + LLM)

| Meyar | Rule-based | LLM | Strategiya |
|-------|-----------|-----|------------|
| KR2.1 Fəal yardım | Response time, rədd edici ifadələr | Semantik qiymətləndirmə | LLM əsaslı, rule siqnallarla kömək edir |
| KR2.2 Tələbatın formalaşdırılması | Təkrarlanan suallar, cavabsız suallar | Dolğunluq qiymətləndirməsi | Hibrid — rule təkrarları tapır, LLM keyfiyyəti yoxlayır |
| KR2.3 Məhsul bilikləri | Rəqəmsal məlumat mövcudluğu | Dəqiqlik və dolğunluq | LLM əsaslı |
| KR2.4 Yönləndirmə/Qeydiyyat | Yönləndirmə açar sözləri | Kontekst analizi | Hibrid |
| KR2.5 Peşəkar davranış | Qadağan olunmuş ifadələr, daxili məlumat sızması | Edge case-lər | Rule əsaslı — dəqiq match HIGH confidence ilə LLM-i atlar |

### Nə vaxt Rule, nə vaxt LLM?

- **Rule HIGH confidence** → LLM çağırılmır (məs: operator "micro donub" dedi → score 0, şübhə yoxdur)
- **Rule MEDIUM confidence** → LLM təsdiqləyir (məs: heç nə tapmadı, amma 100% əmin deyil)
- **Rule LOW confidence** → LLM qərar verir (məs: semantik analiz tələb olunur)

## Quraşdırma

### Tələblər
- Python 3.11+
- Groq API key ([console.groq.com](https://console.groq.com))

### Addımlar
```bash
git clone https://github.com/Fuad-grb/kontakt-home-qa.git
cd kontakt-home-qa
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
# .env faylında GROQ_API_KEY dəyərini yazın
```

### Docker ilə
```bash
docker-compose up qa-pipeline
```

## İstifadə

### Tək transkript qiymətləndirmə
```bash
python main.py --input transcript.json --output result.json
```

### Batch (eval dataset)
```bash
python main.py --input eval/eval_dataset.json --batch --output output/results.json
```

### Evaluation (accuracy ölçmə)
```bash
PYTHONPATH=. python eval/evaluate.py --limit 10
```

## Testlər
```bash
python -m pytest tests/ -v
```

Docker ilə:
```bash
docker-compose --profile test run test
```

## Texniki qərarlar

### LLM seçimi: Groq + Llama 3.3 70B
- **Niyə Groq?** Pulsuz tier, sürətli inference (~0.5s per call)
- **Niyə Llama 3.3 70B?** Azərbaycan dilini daha yaxşı anlayır (8B modeldən fərqli olaraq)
- **Niyə `temperature=0.1`?** Daha deterministik nəticələr — qiymətləndirmə subyektiv olmamalıdır

### Anti-hallucination tədbirləri
- System prompt-da açıq şəkildə "YALNIZ transkriptdə olan məlumatlara əsaslan" qaydası
- Rule-based siqnallar LLM-ə kontekst kimi ötürülür
- JSON response format məcburi edilir (`response_format: json_object`)
- Score 0-3 aralığına clamp edilir

### Prompt idarəetməsi
- Bütün promptlar `prompts/criteria.yaml` faylında saxlanılır
- Koddan ayrıdır — asanlıqla dəyişdirilə bilər

### PII Detection
- Regex əsaslı (ML lazım deyil — formatlar strukturlaşdırılmışdır)
- FIN kodları, kart nömrələri, telefon nömrələri
- False positive azaltma: FIN disambiguation (KONTAKT, SAMSUNG kimi sözlər filtrlənir)

## Qarşılaşılan çətinliklər

1. **Groq rate limiting** — Pulsuz tier limiti var. Exponential backoff retry ilə həll edildi.
2. **KR2.3 meyarı** — Excel faylında yox idi, eval dataset-dən analiz edilərək "Məhsul/xidmət bilikləri" kimi müəyyən edildi.
3. **FIN kod false positive** — 7 simvollu hər söz FIN deyil. Disambiguation funksiyası əlavə edildi.

