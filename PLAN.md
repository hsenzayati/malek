# 🤖 خطة مشروع: نظام تداول ذكي متعدد الوكلاء (Multi-Agent AI Trading System)
## للذهب (XAU/USD) والبيتكوين (BTC/USDT) — تحليل Real-Time — بحد أقصى 100 صفقة/يوم

> **بدون أي API Token لنماذج LLM** (لا Anthropic، لا OpenAI، لا Gemini) — كل النماذج تعمل وتتدرّب **محلياً** على حاسوبك.

---

## 1. ملاءمة العتاد (Hardware Check)

| المكوّن | المواصفات | التقييم |
|---|---|---|
| GPU | RTX 3050 (4GB VRAM) | ✅ كافٍ لتدريب LSTM/TCN صغيرة، تشغيل FinBERT للاستدلال، وتدريب RL (PPO) |
| CPU | Intel i5 الجيل 12 | ✅ ممتاز لـ LightGBM/XGBoost والمعالجة اللحظية |
| RAM | 16GB | ✅ كافٍ بشرط استخدام بيانات Parquet مضغوطة و batching |
| OS | Windows | ✅ ميزة إضافية: يدعم MetaTrader5 Python API لسعر الذهب اللحظي |

**القاعدة الذهبية:** لا نستخدم نماذج ضخمة. نستخدم نماذج صغيرة متخصصة (specialist models) — وهذا فعلياً أفضل للتداول من LLM عام.

---

## 2. مصادر البيانات اللحظية (Real-Time Data) — كلها مجانية

### البيتكوين BTC/USDT
- **Binance WebSocket** (`wss://stream.binance.com`) — بيانات tick لحظية حقيقية، مجانية 100%، بدون مفتاح للبيانات العامة.
- مكتبة: `python-binance` أو `ccxt.pro` (أو `websockets` مباشرة).

### الذهب XAU/USD
- **الخيار الأساسي:** MetaTrader 5 + حساب Demo مجاني (مثل ICMarkets/Exness) عبر مكتبة `MetaTrader5` في بايثون — سعر لحظي حقيقي للذهب، ويعمل فقط على Windows (حاسوبك مناسب تماماً).
- **خيار احتياطي:** عملة **PAXG/USDT** على Binance (ذهب مرمّز يتبع سعر الأونصة) — نفس الـ WebSocket اللحظي.

### الأخبار (للوكيل الإخباري)
- **RSS Feeds:** Investing.com, CoinDesk, Reuters commodities, FXStreet.
- **ForexFactory Calendar** (scraping/JSON): مواعيد الأحداث الاقتصادية (CPI, NFP, قرارات الفيدرالي) — أهم شيء للذهب.
- **CryptoPanic API** (مفتاح مجاني، ليس LLM): أخبار الكريبتو مصنّفة.
- التحليل يتم **محلياً** بنموذج FinBERT (انظر Agent 3).

---

## 3. هيكلية الوكلاء الخمسة (The 5 AI Agents)

```
                    ┌──────────────────────────┐
                    │   Agent 5: المنسّق        │
                    │  Decision Coordinator     │
                    │  (Meta-Learner / Ensemble)│
                    └────────────▲─────────────┘
          ┌──────────────┬──────┴──────┬──────────────┐
          │              │             │              │
   ┌──────┴─────┐ ┌──────┴─────┐ ┌────┴───────┐ ┌────┴───────┐
   │  Agent 1   │ │  Agent 2   │ │  Agent 3   │ │  Agent 4   │
   │ محلل السوق │ │ وكيل التداول│ │ محلل الأخبار│ │ مدير المخاطر│
   │ Market     │ │ Trading    │ │ News &     │ │ Risk       │
   │ Analyst    │ │ Executor   │ │ Sentiment  │ │ Manager    │
   └──────▲─────┘ └──────▲─────┘ └─────▲──────┘ └─────▲──────┘
          │              │             │              │
   ┌──────┴──────────────┴─────────────┴──────────────┴──────┐
   │        Data Bus (Redis Pub/Sub أو asyncio queues)        │
   │   Binance WS │ MT5 Ticks │ RSS/News │ Economic Calendar  │
   └──────────────────────────────────────────────────────────┘
```

### 🔵 Agent 1 — محلل السوق (Market Analyst)
- **المهمة:** توقّع اتجاه السعر القصير المدى (صعود/هبوط/محايد) + قوة الإشارة.
- **النماذج (تتدرّب على حاسوبك):**
  - **LightGBM** (CPU): تصنيف الاتجاه من ~60 مؤشراً فنياً (RSI, MACD, Bollinger, ATR, OBV, VWAP, EMA crossovers…) عبر مكتبة `pandas-ta`.
  - **LSTM/TCN صغيرة** (GPU, PyTorch): توقّع تسلسلي على شموع 1m/5m/15m (نموذج < 5M باراميتر — يتدرّب في دقائق على RTX 3050).
  - دمج النموذجين = إشارة نهائية بثقة (probability).
- **المخرَج:** `{direction: long/short/flat, confidence: 0–1, horizon: 15m}` كل شمعة.

### 🟢 Agent 2 — وكيل التداول (Trading Executor)
- **المهمة:** تحويل الإشارات إلى قرارات تنفيذ: دخول/خروج، حجم الصفقة، Stop-Loss/Take-Profit.
- **النموذج:** **Reinforcement Learning — PPO** عبر `stable-baselines3` على بيئة `gymnasium` مخصصة:
  - State: إشارات Agent 1 + 3 + وضع المحفظة + التقلب الحالي.
  - Action: {buy, sell, hold, close} + حجم الصفقة.
  - Reward: العائد المعدّل بالمخاطرة (Sharpe-based) − عمولات − انزلاق سعري.
- يتدرّب على بيانات تاريخية (backtesting env) ثم يُصقَل على paper trading.

### 🟠 Agent 3 — محلل الأخبار (News & Sentiment Analyst)
- **المهمة:** قراءة الأخبار لحظياً وتقدير تأثيرها على الذهب والبيتكوين.
- **النماذج المحلية (بدون أي API لـ LLM):**
  - **FinBERT** (`ProsusAI/finbert` من Hugging Face — يُحمَّل مرة واحدة ويعمل offline): تصنيف المشاعر المالية (إيجابي/سلبي/محايد). حجمه ~440MB ويعمل بسلاسة على 4GB VRAM.
  - **نموذج تأثير مدرَّب محلياً (LightGBM):** يتعلم من التاريخ: (نوع الحدث الاقتصادي + المفاجأة مقابل التوقع) → حركة السعر خلال 15/60 دقيقة.
  - **مرشّح الأحداث:** قبل CPI/NFP/FOMC بـ 15 دقيقة → إشارة "خطر عالٍ" توقف التداول تلقائياً.
- **المخرَج:** `{sentiment_gold: -1..+1, sentiment_btc: -1..+1, event_risk: low/high}`.

### 🔴 Agent 4 — مدير المخاطر (Risk Manager) ← *اختياري رقم 1*
- **المهمة:** حماية رأس المال — له **حق الفيتو** على أي صفقة.
- **القواعد + نموذج ML:**
  - عدّاد الصفقات: **حد أقصى 100 صفقة/يوم** (hard limit) + حد أقصى للخسارة اليومية (مثلاً −3%) → إيقاف تلقائي.
  - حجم الصفقة: Kelly Criterion مخفَّف / نسبة مخاطرة ثابتة 0.5–1% لكل صفقة.
  - **نموذج توقع التقلب (GARCH أو LightGBM):** يوسّع الـ SL ويقلّص الحجم عند التقلب العالي.
  - كشف الشذوذ (Isolation Forest): flash crashes، فجوات سيولة → إغلاق فوري.

### 🟣 Agent 5 — المنسّق (Decision Coordinator / Meta-Agent) ← *اختياري رقم 2*
- **المهمة:** الدمج النهائي — يستقبل مخرجات الوكلاء الأربعة ويصدر القرار النهائي.
- **النموذج:** **Stacking Meta-Learner** (Logistic Regression أو LightGBM صغير) يتعلّم أوزان الثقة في كل وكيل حسب حالة السوق (trending/ranging/news-driven).
- **منطق العمل:**
  1. Agent 3 يقول event_risk=high؟ → لا تداول.
  2. Agent 1 و Agent 3 متفقان بثقة > عتبة؟ → مرّر لـ Agent 2.
  3. Agent 2 يقترح صفقة → Agent 4 يوافق أو يرفض.
  4. سجلّ كل قرار مع تعليل (audit log) للتحسين لاحقاً.

---

## 4. البنية التقنية (Tech Stack)

| الطبقة | الأداة | لماذا |
|---|---|---|
| اللغة | Python 3.11 | النظام البيئي الأكمل للتداول وML |
| بيانات لحظية | `python-binance` (WS) + `MetaTrader5` | tick حقيقي مجاني |
| مؤشرات فنية | `pandas-ta` | 130+ مؤشر جاهز |
| ML كلاسيكي | `lightgbm`, `scikit-learn` | سريع على CPU، دقيق على بيانات جدولية |
| Deep Learning | `torch` (CUDA 12.x) | LSTM/TCN على الـ RTX 3050 |
| RL | `stable-baselines3` + `gymnasium` | PPO جاهز وموثوق |
| NLP محلي | `transformers` + FinBERT | تحليل مشاعر offline |
| Backtesting | `vectorbt` | أسرع backtester (vectorized) |
| تواصل الوكلاء | `asyncio` queues (بسيط) أو Redis Pub/Sub | لحظي وخفيف |
| تخزين | Parquet (تاريخي) + SQLite (صفقات/قرارات) | خفيف على 16GB RAM |
| لوحة مراقبة | Streamlit dashboard | مراقبة حية للإشارات والأرباح |

---

## 5. مراحل التنفيذ (Roadmap)

### المرحلة 0 — التجهيز (يوم–يومان)
- [ ] تثبيت Python 3.11 + CUDA + PyTorch + المكتبات.
- [ ] فتح حساب Binance (Testnet) + حساب MT5 Demo.
- [ ] هيكلة المشروع (انظر §6).

### المرحلة 1 — خط أنابيب البيانات (أسبوع)
- [ ] `data/collector_binance.py`: WebSocket → ticks → شموع 1m → Parquet.
- [ ] `data/collector_mt5.py`: نفس الشيء للذهب.
- [ ] `data/collector_news.py`: RSS + التقويم الاقتصادي → SQLite.
- [ ] تنزيل بيانات تاريخية سنتين+ (Binance klines API + MT5 history) للتدريب.

### المرحلة 2 — تدريب النماذج (أسبوعان)
- [ ] هندسة الميزات (features) + labeling بطريقة **Triple-Barrier** (من كتاب Advances in Financial ML).
- [ ] تدريب LightGBM + LSTM لـ Agent 1 (تدريب منفصل للذهب والبيتكوين).
- [ ] تدريب FinBERT pipeline + نموذج تأثير الأخبار لـ Agent 3.
- [ ] تقييم صارم: **Walk-Forward Validation** (لا تثق أبداً بنتيجة train/test عشوائي في التداول!).

### المرحلة 3 — بيئة RL ووكيل التداول (أسبوعان)
- [ ] بناء `TradingEnv` (gymnasium) بعمولات وانزلاق واقعيين.
- [ ] تدريب PPO لـ Agent 2 (ليلة تدريب على RTX 3050 كافية لكل جولة).
- [ ] بناء Agent 4 (قواعد + GARCH) و Agent 5 (meta-learner).

### المرحلة 4 — الدمج والـ Backtesting (أسبوع)
- [ ] تشغيل النظام كاملاً على بيانات تاريخية.
- [ ] المقاييس المستهدفة قبل المتابعة: Sharpe > 1.0، Max Drawdown < 15%، Profit Factor > 1.3.

### المرحلة 5 — Paper Trading (شهر كامل — إجباري)
- [ ] تشغيل لحظي حقيقي على Binance Testnet + MT5 Demo.
- [ ] مراقبة عبر Streamlit dashboard + إعادة تدريب أسبوعية (retraining pipeline).
- [ ] لا أموال حقيقية قبل شهر أخضر من الورق.

### المرحلة 6 — التشغيل الحقيقي (تدريجي)
- [ ] مبلغ صغير جداً أولاً، مع كل حدود Agent 4 مفعّلة.

---

## 6. هيكل المشروع المقترح

```
malek/
├── config/
│   └── settings.yaml          # الرموز، الحدود (100 صفقة/يوم)، عتبات الثقة
├── data/
│   ├── collectors/            # binance_ws.py, mt5_ticks.py, news_rss.py
│   ├── historical/            # Parquet files
│   └── features.py            # هندسة الميزات المشتركة
├── agents/
│   ├── agent1_market.py       # LightGBM + LSTM
│   ├── agent2_trader.py       # PPO (stable-baselines3)
│   ├── agent3_news.py         # FinBERT + event impact model
│   ├── agent4_risk.py         # حدود + GARCH + anomaly detection
│   └── agent5_coordinator.py  # meta-learner + منطق القرار
├── training/
│   ├── train_market.py
│   ├── train_rl.py
│   └── walk_forward.py
├── backtest/
│   └── run_backtest.py        # vectorbt
├── execution/
│   ├── binance_exec.py        # تنفيذ فعلي/testnet
│   └── mt5_exec.py
├── dashboard/
│   └── app.py                 # Streamlit
├── main.py                    # asyncio orchestrator — يشغّل كل الوكلاء
└── requirements.txt
```

---

## 7. قيد الـ 100 صفقة/يوم — التطبيق العملي

- عدّاد يومي في Agent 4 يُصفَّر عند 00:00 UTC، مخزَّن في SQLite (يصمد أمام إعادة التشغيل).
- Agent 5 يرفع عتبة الثقة تدريجياً كلما اقترب العدّاد من الحد (مثلاً بعد 70 صفقة يقبل فقط إشارات بثقة > 0.8) — حتى تُصرَف "الميزانية" على أفضل الفرص.
- التوزيع المقترح: ~60 للبيتكوين (سوق 24/7) و ~40 للذهب (جلسات لندن/نيويورك).

---

## 8. تحذيرات مهمة وواقعية

1. **لا يوجد نموذج يضمن الربح.** حتى الصناديق الكبرى تخسر. الهدف الواقعي: edge صغير + إدارة مخاطر صارمة.
2. **أخطر عدو هو Overfitting:** نموذج يبدو عبقرياً على التاريخ ويخسر مباشرة. الحل: Walk-forward + بيانات out-of-sample لا تلمسها أبداً أثناء التطوير.
3. **العمولات والانزلاق** تقتل استراتيجيات التداول عالي التردد — 100 صفقة/يوم يعني أن كل صفقة يجب أن تربح أكثر من التكاليف. احسبها في الـ backtest من اليوم الأول.
4. **ابدأ بالورق (Paper Trading) دائماً** — شهر كامل على الأقل.
5. تدريب FinBERT fine-tuning كامل قد يضغط على 4GB VRAM — استخدمه كما هو (inference) أو درّب طبقة التصنيف الأخيرة فقط.

---

## 9. الخطوة التالية

بعد الموافقة على هذه الخطة، نبدأ بالمرحلة 0 و 1: بناء هيكل المشروع + جامعات البيانات (data collectors) لـ Binance و MT5 والأخبار.
