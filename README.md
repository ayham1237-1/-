//@version=5
indicator("SMC Brain - APEX v13.0 Research [Integrity]", overlay=true, max_lines_count=500, max_labels_count=500, max_boxes_count=500, max_bars_back=500, dynamic_requests=true)

// =============================================================================
// HELPER FUNCTIONS
// =============================================================================
safe_div(num, den, fallback) =>
    den != 0 and not na(den) ? num / den : fallback

safe_max(a, b) =>
    na(a) ? b : na(b) ? a : math.max(a, b)

safe_min(a, b) =>
    na(a) ? b : na(b) ? a : math.min(a, b)

age_ok(age, max_age) =>
    not na(age) and age >= 0 and age <= max_age

// =============================================================================
// CONFIGURATION
// =============================================================================
grp_profile = "البروفايل التكيفي"
adaptive_profile = input.string("ذهب/بتكوين ترند", "بروفايل السوق", options=["عام", "ذهب/بتكوين ترند", "فوركس سيولة", "قناص صارم", "فوري سريع"], group=grp_profile)
use_adaptive_profile = input.bool(true, "تفعيل البروفايل التكيفي", group=grp_profile)
show_decision_stage = input.bool(true, "إظهار مراحل القرار", group=grp_profile)

grp_disp = "العرض"
show_visuals = input.bool(true, "إظهار العناصر البصرية", group=grp_disp)
show_signals = input.bool(true, "إظهار الإشارات", group=grp_disp)
show_panel = input.bool(true, "إظهار لوحة المعلومات", group=grp_disp)
show_regime_bg = input.bool(false, "إظهار خلفية حالة السوق", group=grp_disp)
show_debug_marks = input.bool(false, "إظهار علامات التشخيص", group=grp_disp)
show_killzone_bg = input.bool(false, "إظهار خلفية الكيلزون", group=grp_disp)

grp_sig = "الإشارات"
enable_quick = input.bool(true, "تفعيل إشارات Quick", group=grp_sig)
enable_strong = input.bool(true, "تفعيل إشارات Strong", group=grp_sig)
enable_hyper = input.bool(true, "تفعيل إشارات Hyper", group=grp_sig)
quick_score_min = input.int(45, "أقل درجة Quick", minval=1, maxval=100, group=grp_sig)
strong_score_min = input.int(68, "أقل درجة Strong", minval=1, maxval=100, group=grp_sig)
hyper_score_min = input.int(84, "أقل درجة Hyper", minval=1, maxval=100, group=grp_sig)

grp_strict = "صرامة الإشارات"
quick_require_sweep = input.bool(true, "Quick يحتاج Sweep حديث", group=grp_strict)
quick_require_trend = input.bool(true, "Quick يحتاج اتجاه SMA20", group=grp_strict)
strong_require_sequence = input.bool(true, "Strong يحتاج تسلسل Sweep→BOS", group=grp_strict)
strong_require_fib = input.bool(true, "Strong يحتاج OTE", group=grp_strict)
strong_require_session = input.bool(false, "Strong يحتاج جلسة لندن/نيويورك", group=grp_strict)
strong_require_htf = input.bool(true, "Strong يحتاج اتجاه HTF", group=grp_strict)
hyper_require_physics = input.bool(true, "Hyper يحتاج فيزياء السعر", group=grp_strict)
hyper_require_premium_discount = input.bool(true, "Hyper يحتاج Discount/Premium", group=grp_strict)
hyper_require_idm = input.bool(true, "Hyper يحتاج Inducement (IDM)", group=grp_strict)
memory_max_age = input.int(12, "أقصى عمر الذاكرة (شموع)", minval=1, maxval=60, group=grp_strict)
cooldown_bars = input.int(7, "شموع منع تكرار Quick", minval=1, maxval=60, group=grp_strict)

grp_ms = "هيكل السوق"
ms_left = input.int(3, "شموع يسار السوينغ", minval=1, maxval=20, group=grp_ms)
ms_right = input.int(3, "شموع يمين السوينغ", minval=1, maxval=20, group=grp_ms)
internal_left = input.int(1, "شموع يسار الهيكل الداخلي", minval=1, maxval=5, group=grp_ms)
internal_right = input.int(1, "شموع يمين الهيكل الداخلي", minval=1, maxval=5, group=grp_ms)
bos_body_atr = input.float(0.45, "أقل جسم BOS نسبة ATR", minval=0.0, maxval=3.0, step=0.05, group=grp_ms)
choch_body_atr = input.float(0.15, "أقل جسم CHoCH نسبة ATR", minval=0.0, maxval=2.0, step=0.05, group=grp_ms)

grp_liq = "السيولة"
liq_pivot_lookback = input.int(40, "ذاكرة قمم/قيعان السيولة", minval=5, maxval=200, group=grp_liq)
sweep_atr_depth_min = input.float(0.03, "أقل عمق Sweep نسبة ATR", minval=0.0, maxval=1.0, step=0.01, group=grp_liq)
sweep_reject_min = input.float(0.35, "أقل رفض بالذيل للـ Sweep", minval=0.0, maxval=1.0, step=0.05, group=grp_liq)
eq_atr_tol = input.float(0.06, "سماحية التساوي نسبة ATR", minval=0.01, maxval=0.30, step=0.01, group=grp_liq)

grp_fvg = "FVG + Inversion"
fvg_min_size = input.float(0.20, "أقل حجم FVG نسبة ATR", minval=0.0, maxval=3.0, step=0.05, group=grp_fvg)
fvg_require_middle_displacement = input.bool(true, "اشتراط اندفاع الشمعة الوسطى", group=grp_fvg)
fvg_middle_body_atr = input.float(0.35, "جسم الشمعة الوسطى نسبة ATR", minval=0.0, maxval=3.0, step=0.05, group=grp_fvg)
fvg_mitigation_mode = input.string("Touch", "طريقة إلغاء FVG", options=["Touch", "Close"], group=grp_fvg)
use_inversion_fvg = input.bool(true, "تفعيل Inversion FVG", group=grp_fvg)

grp_ob = "الأوردر بلوك"
ob_displacement = input.float(1.15, "اندفاع OB نسبة ATR", minval=0.1, maxval=5.0, step=0.05, group=grp_ob)
ob_use_body = input.bool(false, "استخدام جسم OB بدل الذيل", group=grp_ob)
ob_mitigation = input.bool(true, "رسم مناطق OB", group=grp_ob)
ob_mitigation_mode = input.string("Touch", "طريقة إلغاء OB", options=["Touch", "Close"], group=grp_ob)
ob_quality_min = input.int(45, "أقل جودة OB", minval=0, maxval=100, group=grp_ob)
max_ob_zones = input.int(5, "أقصى عدد مناطق OB", minval=1, maxval=10, group=grp_ob)

grp_fib = "Premium / Discount"
fib_zone_start = input.float(0.618, "بداية منطقة OTE", minval=0.0, maxval=1.0, step=0.001, group=grp_fib)
fib_zone_end = input.float(0.790, "نهاية منطقة OTE", minval=0.0, maxval=1.0, step=0.001, group=grp_fib)
require_discount_for_longs = input.bool(false, "اشتراط Discount لشراء Strong", group=grp_fib)
require_premium_for_shorts = input.bool(false, "اشتراط Premium لبيع Strong", group=grp_fib)
fib_style = input.string("Lines", "شكل عرض OTE", options=["Lines", "Box", "None"], group=grp_fib)
fib_line_color = input.color(color.gray, "لون خط OTE", group=grp_fib)
fib_fill_color = input.color(color.new(color.gray, 90), "لون تعبئة OTE", group=grp_fib)
fib_line_width = input.int(1, "سماكة خط OTE", minval=1, maxval=4, group=grp_fib)
fib_line_style = input.string("Dashed", "نوع خط OTE", options=["Solid", "Dashed", "Dotted"], group=grp_fib)

grp_sess = "الجلسات + Killzones"
use_session_filter = input.bool(false, "استخدام فلتر الجلسات", group=grp_sess)
use_killzone_logic = input.bool(true, "تفعيل منطق Killzone (AMD)", group=grp_sess)
session_asia = input.session("0000-0800", "جلسة آسيا", group=grp_sess)
session_london = input.session("0800-1600", "جلسة لندن", group=grp_sess)
session_ny = input.session("1300-2100", "جلسة نيويورك", group=grp_sess)

grp_mom = "الزخم"
momentum_method = input.string("Hybrid", "طريقة الزخم", options=["RSI", "Original", "Hybrid"], group=grp_mom)
rsi_len = input.int(14, "طول RSI", minval=2, maxval=100, group=grp_mom)
rsi_bull_thresh = input.float(58.0, "عتبة RSI للشراء", minval=1.0, maxval=99.0, group=grp_mom)
rsi_bear_thresh = input.float(42.0, "عتبة RSI للبيع", minval=1.0, maxval=99.0, group=grp_mom)
vol_factor = input.float(1.05, "مضاعف الحجم مقابل SMA20", minval=0.1, maxval=5.0, step=0.05, group=grp_mom)
mom_lookback = input.int(10, "فترة الزخم الأصلي", minval=2, maxval=80, group=grp_mom)
momentum_threshold = input.float(60.0, "عتبة الزخم الأصلي", minval=50.0, maxval=95.0, step=0.5, group=grp_mom)

grp_htf = "الفريم الأعلى HTF"
use_htf_trend = input.bool(true, "استخدام اتجاه HTF", group=grp_htf)
htf_tf = input.timeframe("240", "الفريم الأعلى", group=grp_htf)
htf_method = input.string("EMA", "طريقة HTF", options=["EMA", "Structure"], group=grp_htf)
htf_ema_len = input.int(50, "طول EMA HTF", minval=5, maxval=300, group=grp_htf)

grp_reg = "حالة السوق"
use_regime = input.bool(true, "تفعيل كشف حالة السوق ADX", group=grp_reg)
adx_len = input.int(14, "طول ADX", minval=5, maxval=50, group=grp_reg)
adx_trend_thresh = input.float(23.0, "عتبة ADX للترند", minval=5.0, maxval=60.0, step=0.5, group=grp_reg)
adx_range_thresh = input.float(18.0, "عتبة ADX للرنج", minval=5.0, maxval=40.0, step=0.5, group=grp_reg)
use_volatility_regime = input.bool(true, "استخدام رتبة ATR", group=grp_reg)
atr_percentile_len = input.int(100, "فترة رتبة ATR", minval=20, maxval=300, group=grp_reg)
atr_expansion_rank = input.float(60.0, "رتبة ATR للتمدد", minval=1.0, maxval=99.0, step=1.0, group=grp_reg)
atr_extreme_rank = input.float(90.0, "رتبة ATR الخطرة", minval=50.0, maxval=99.0, step=1.0, group=grp_reg)

grp_phys = "فيزياء السعر"
physics_len = input.int(10, "فترة فيزياء السعر", minval=3, maxval=80, group=grp_phys)
velocity_min = input.float(0.08, "أقل سرعة نسبة ATR", minval=0.0, maxval=2.0, step=0.01, group=grp_phys)
efficiency_min = input.float(0.25, "أقل كفاءة حركة", minval=0.0, maxval=1.0, step=0.01, group=grp_phys)
pressure_min = input.float(0.05, "أقل ضغط حجم موقّع", minval=0.0, maxval=2.0, step=0.01, group=grp_phys)

grp_risk = "إدارة المخاطر"
use_risk_filter = input.bool(true, "استخدام فلتر RR الحقيقي", group=grp_risk)
rr_min = input.float(2.0, "أقل نسبة RR", minval=1.0, maxval=6.0, step=0.25, group=grp_risk)
sl_atr_buffer = input.float(0.50, "هامش وقف الخسارة ATR", minval=0.0, maxval=3.0, step=0.05, group=grp_risk)

grp_cost = "نموذج التكاليف (Execution Cost Model)"
spread_ticks = input.float(2.0, "السبريد (ticks)", minval=0.0, step=0.1, group=grp_cost)
commission_ticks = input.float(1.0, "العمولة لكل جانب (ticks)", minval=0.0, step=0.1, group=grp_cost)
slippage_ticks = input.float(1.0, "الانزلاق المقدر (ticks)", minval=0.0, step=0.1, group=grp_cost)

grp_filter = "فلاتر التحليل"
block_short_when_htf_bull = input.bool(true, "منع بيع إذا HTF صاعد", group=grp_filter)
block_long_when_htf_bear = input.bool(false, "منع شراء إذا HTF هابط", group=grp_filter)
use_score_bias_filter = input.bool(true, "منع الإشارة إذا الدرجة المعاكسة أقوى", group=grp_filter)
score_bias_gap = input.int(8, "فرق الدرجة المطلوب", minval=0, maxval=50, group=grp_filter)
show_blocked_signals = input.bool(false, "إظهار الإشارات الممنوعة", group=grp_filter)
use_hard_trend_guard = input.bool(true, "حارس ترند صارم", group=grp_filter)
trend_guard_ema_len = input.int(200, "EMA حارس الترند", minval=20, maxval=500, group=grp_filter)
trend_guard_slope_min = input.float(0.015, "أقل ميل HTF", minval=0.0, maxval=1.0, step=0.005, group=grp_filter)
no_trade_when_range = input.bool(true, "منع الإشارات المؤكدة في الرنج", group=grp_filter)
no_trade_when_extreme_atr = input.bool(true, "منع الإشارات عند ATR خطر", group=grp_filter)

grp_rt = "الإشارات الفورية"
use_realtime_signals = input.bool(true, "تفعيل الإشارات الفورية", group=grp_rt)
show_realtime_signals = input.bool(true, "رسم الإشارات الفورية", group=grp_rt)
realtime_alerts = input.bool(true, "تفعيل تنبيهات فورية", group=grp_rt)
realtime_mode = input.string("درجة مباشرة", "طريقة الإشارة الفورية", options=["كسر سيولة", "درجة مباشرة", "كلاهما"], group=grp_rt)
show_realtime_on_history = input.bool(true, "إظهار فورية تاريخيًا", group=grp_rt)
realtime_lookback = input.int(12, "فترة كسر السيولة الفورية", minval=3, maxval=80, group=grp_rt)
realtime_score_min = input.int(55, "أقل درجة فورية", minval=1, maxval=100, group=grp_rt)
realtime_require_htf = input.bool(false, "فورية تحتاج HTF", group=grp_rt)
realtime_require_momentum = input.bool(false, "فورية تحتاج زخم", group=grp_rt)
realtime_require_rejection = input.bool(false, "فورية تحتاج رفض", group=grp_rt)

grp_mtf = "الفريمات المتعددة MTF"
use_mtf_dashboard = input.bool(true, "تفعيل MTF Dashboard", group=grp_mtf)
mtf_tf1 = input.timeframe("15", "فريم 1", group=grp_mtf)
mtf_tf2 = input.timeframe("60", "فريم 2", group=grp_mtf)
mtf_tf3 = input.timeframe("240", "فريم 3", group=grp_mtf)

grp_volprof = "بروفايل الحجم (Rolling Volume Node)"
use_vol_profile = input.bool(true, "تفعيل تقريب العقدة الحجمية", group=grp_volprof)
vp_lookback = input.int(50, "فترة العقدة الحجمية", minval=20, maxval=200, group=grp_volprof)
vp_bins = input.int(10, "عدد مستويات العقدة", minval=5, maxval=20, group=grp_volprof)

grp_research = "APEX Research Engine"
research_enabled = input.bool(true, "تفعيل Research Engine", group=grp_research)
entry_mode = input.string("Next Open", "نمط الدخول", options=["Signal Close", "Next Open"], group=grp_research)
max_trade_bars = input.int(48, "أقصى مدة صفقة (شموع)", minval=1, maxval=500, group=grp_research)
ambiguous_policy = input.string("Conservative", "سياسة الشمعة الغامضة", options=["Conservative", "Optimistic", "Ignore"], group=grp_research)
show_research_dashboard = input.bool(false, "إظهار لوحة Research", group=grp_research)
research_max_active = input.int(20, "أقصى صفقات نشطة", minval=5, maxval=100, group=grp_research)
research_max_closed = input.int(500, "أقصى صفقات مغلقة", minval=100, maxval=2000, group=grp_research)
show_equity_curve = input.bool(true, "رسم منحنى Equity (R-based)", group=grp_research)
export_alerts = input.bool(true, "تصدير JSON للـCollector", group=grp_research)

// =============================================================================
// V13 NEW INPUT GROUPS
// =============================================================================

// --- Step 5: Intrabar Execution Engine ---
grp_ib = "Intrabar Execution Engine (V13)"
use_intrabar_engine = input.bool(true, "تفعيل الحسم داخل الشمعة", group=grp_ib)
intrabar_tf = input.timeframe("1", "الفريم الأدنى للحسم", group=grp_ib)

// --- Step 6: MTF Architecture Engine ---
grp_mtf_arch = "MTF Architecture Engine (V13)"
use_mtf_architecture = input.bool(true, "تفعيل محرك MTF المعماري", group=grp_mtf_arch)
mtf_macro_tf = input.timeframe("D", "فريم الماكرو (Bias الأعلى)", group=grp_mtf_arch)
mtf_require_alignment = input.bool(false, "اشتراط توافق MTF للإشارات المؤكدة", group=grp_mtf_arch)
mtf_alignment_min = input.int(2, "أقل عدد فريمات متوافقة (من 4)", minval=1, maxval=4, group=grp_mtf_arch)

// --- Step 8: Anti-Overfitting & Stability ---
grp_stability = "Anti-Overfitting & Stability Engine (V13)"
use_stability_engine = input.bool(true, "تفعيل محرك الثبات", group=grp_stability)
stability_min_trades = input.int(30, "أقل عدد صفقات للتقييم", minval=10, maxval=500, group=grp_stability)
stability_recent_len = input.int(50, "نافذة الأداء الحديثة", minval=10, maxval=500, group=grp_stability)
show_stability_dashboard = input.bool(true, "إظهار لوحة الثبات", group=grp_stability)

// --- Step 9: Research Export Bridge ---
grp_export = "Research Export Bridge (V13)"
use_export_bridge = input.bool(true, "تفعيل جسر التصدير", group=grp_export)
export_closed_trades = input.bool(true, "تصدير الصفقات المغلقة", group=grp_export)
export_format = input.string("JSON", "صيغة التصدير", options=["JSON", "CSV"], group=grp_export)
export_include_snapshots = input.bool(true, "تضمين Features الإضافية", group=grp_export)
export_on_confirmed_only = input.bool(false, "التصدير بعد تأكيد الشمعة فقط", group=grp_export)

// --- Step 10: Production Readiness Gate ---
grp_prod = "Production Readiness Gate (V13 Final)"
use_production_gate = input.bool(true, "تفعيل بوابة الجاهزية للإنتاج", group=grp_prod)
prod_min_trades = input.int(100, "أقل عدد صفقات للإنتاج", minval=30, maxval=1000, group=grp_prod)
prod_min_pf = input.float(1.30, "أقل Profit Factor", minval=1.0, maxval=3.0, step=0.05, group=grp_prod)
prod_min_expectancy = input.float(0.10, "أقل Expectancy (R)", minval=0.0, maxval=1.0, step=0.01, group=grp_prod)
prod_max_dd = input.float(10.0, "أقصى Drawdown مقبول (R)", minval=1.0, maxval=50.0, step=0.5, group=grp_prod)
show_production_dashboard = input.bool(true, "إظهار لوحة الجاهزية", group=grp_prod)

// =============================================================================
// ADAPTIVE PROFILE
// =============================================================================
int profile_quick_score_min = quick_score_min
int profile_strong_score_min = strong_score_min
int profile_hyper_score_min = hyper_score_min
int profile_realtime_score_min = realtime_score_min

if use_adaptive_profile
    profile_quick_score_min := adaptive_profile == "ذهب/بتكوين ترند" ? 48 : adaptive_profile == "فوركس سيولة" ? 44 : adaptive_profile == "قناص صارم" ? 60 : adaptive_profile == "فوري سريع" ? 38 : quick_score_min
    profile_strong_score_min := adaptive_profile == "ذهب/بتكوين ترند" ? 62 : adaptive_profile == "فوركس سيولة" ? 60 : adaptive_profile == "قناص صارم" ? 74 : adaptive_profile == "فوري سريع" ? 56 : strong_score_min
    profile_hyper_score_min := adaptive_profile == "ذهب/بتكوين ترند" ? 80 : adaptive_profile == "فوركس سيولة" ? 78 : adaptive_profile == "قناص صارم" ? 88 : adaptive_profile == "فوري سريع" ? 72 : hyper_score_min
    profile_realtime_score_min := adaptive_profile == "ذهب/بتكوين ترند" ? 52 : adaptive_profile == "فوركس سيولة" ? 50 : adaptive_profile == "قناص صارم" ? 66 : adaptive_profile == "فوري سريع" ? 42 : realtime_score_min

// =============================================================================
// COLORS
// =============================================================================
color c_bg_dark = #0B0F14
color c_panel = #111820
color c_border = #202A35
color c_text_primary = #E6EDF3
color c_text_secondary = #8B98A5
color c_bull = #26A69A
color c_bull_dim = color.new(#26A69A, 70)
color c_bear = #EF5350
color c_bear_dim = color.new(#EF5350, 70)
color c_quick = #00BCD4
color c_strong = #F44336
color c_hyper = #9C27B0
color c_warning = #F2C94C
color c_accent = #5B8DEF
color c_neutral = color.gray
color c_regime_trend = color.new(#26A69A, 91)
color c_regime_range = color.new(#F2C94C, 93)
color c_regime_extreme = color.new(#EF5350, 94)
color c_asia = color.new(#2196F3, 85)
color c_london = color.new(#FF9800, 85)
color c_ny = color.new(#4CAF50, 85)

// =============================================================================
// CORE CALCULATIONS
// =============================================================================
float atr_val = ta.atr(14)
float safe_atr = na(atr_val) or atr_val <= 0.0 ? math.max(syminfo.mintick * 10, 0.0001) : atr_val
float body = math.abs(close - open)
float candle_range = math.max(high - low, syminfo.mintick)
float upper_wick = high - math.max(open, close)
float lower_wick = math.min(open, close) - low
float bull_rejection = candle_range > 0 ? lower_wick / candle_range : 0.0
float bear_rejection = candle_range > 0 ? upper_wick / candle_range : 0.0
bool confirmed = barstate.isconfirmed
float vol_sma = ta.sma(volume, 20)
float vol_ratio = safe_div(volume, vol_sma, 1.0)
bool vol_ok = vol_sma > 0 and volume > vol_sma * vol_factor
bool in_session_asia = not na(time(timeframe.period, session_asia))
bool in_session_london = not na(time(timeframe.period, session_london))
bool in_session_ny = not na(time(timeframe.period, session_ny))
bool is_friday_evening = dayofweek == dayofweek.friday and hour >= 16
bool is_monday_morning = dayofweek == dayofweek.monday and hour < 2
bool day_ok = not is_friday_evening and not is_monday_morning
bool session_ok = day_ok and (not use_session_filter or in_session_london or in_session_ny)
bool strong_session_ok = day_ok and (not strong_require_session or in_session_london or in_session_ny)
float sma20 = ta.sma(close, 20)
float sma50 = ta.sma(close, 50)
bool trend_up = close > sma50
bool trend_down = close < sma50

// =============================================================================
// HTF ENGINE
// =============================================================================
[htf_close_raw, htf_ema_raw, htf_guard_ema_raw, htf_ph_raw, htf_pl_raw, htf_atr_raw] = request.security(
     syminfo.tickerid,
     htf_tf,
     [close, ta.ema(close, htf_ema_len), ta.ema(close, trend_guard_ema_len), ta.pivothigh(high, 3, 3), ta.pivotlow(low, 3, 3), ta.atr(14)],
     lookahead=barmerge.lookahead_off,
     gaps=barmerge.gaps_off)

float htf_close = nz(htf_close_raw[1])
float htf_ema = nz(htf_ema_raw[1])
float htf_guard_ema = nz(htf_guard_ema_raw[1])
float htf_ph = htf_ph_raw[1]
float htf_pl = htf_pl_raw[1]
float htf_atr = nz(htf_atr_raw[1])
bool htf_candle_confirmed = not na(htf_close_raw[1]) and not na(htf_close_raw[2])
float htf_guard_slope = safe_div(htf_guard_ema - htf_guard_ema[1], htf_atr, 0.0)
bool htf_power_bull = use_htf_trend and htf_candle_confirmed and htf_close > htf_guard_ema and htf_guard_slope > trend_guard_slope_min
bool htf_power_bear = use_htf_trend and htf_candle_confirmed and htf_close < htf_guard_ema and htf_guard_slope < -trend_guard_slope_min

bool htf_trend_up = not use_htf_trend
bool htf_trend_down = not use_htf_trend
if use_htf_trend and htf_candle_confirmed
    if htf_method == "EMA"
        htf_trend_up := htf_close > htf_ema
        htf_trend_down := htf_close < htf_ema
    else
        htf_trend_up := not na(htf_ph) and not na(htf_ph[1]) and htf_ph > htf_ph[1]
        htf_trend_down := not na(htf_pl) and not na(htf_pl[1]) and htf_pl < htf_pl[1]

bool strong_htf_bull_ok = not strong_require_htf or htf_trend_up
bool strong_htf_bear_ok = not strong_require_htf or htf_trend_down

// =============================================================================
// MARKET STRUCTURE
// =============================================================================
var float[] swing_highs = array.new<float>()
var int[] swing_high_bars = array.new<int>()
var float[] swing_lows = array.new<float>()
var int[] swing_low_bars = array.new<int>()

float swing_high_val = ta.pivothigh(high, ms_left, ms_right)
float swing_low_val = ta.pivotlow(low, ms_left, ms_right)

if confirmed
    if not na(swing_high_val)
        array.unshift(swing_highs, swing_high_val)
        array.unshift(swing_high_bars, bar_index - ms_right)
        if array.size(swing_highs) > liq_pivot_lookback
            array.pop(swing_highs)
            array.pop(swing_high_bars)
    if not na(swing_low_val)
        array.unshift(swing_lows, swing_low_val)
        array.unshift(swing_low_bars, bar_index - ms_right)
        if array.size(swing_lows) > liq_pivot_lookback
            array.pop(swing_lows)
            array.pop(swing_low_bars)

float nearest_swing_high = array.size(swing_highs) > 0 ? array.get(swing_highs, 0) : na
float nearest_swing_low = array.size(swing_lows) > 0 ? array.get(swing_lows, 0) : na
float structure_swing_high = array.size(swing_highs) > 1 ? array.get(swing_highs, 1) : nearest_swing_high
float structure_swing_low = array.size(swing_lows) > 1 ? array.get(swing_lows, 1) : nearest_swing_low

float internal_high = ta.pivothigh(high, internal_left, internal_right)
float internal_low = ta.pivotlow(low, internal_left, internal_right)
var float last_internal_high = na
var float last_internal_low = na
if confirmed
    if not na(internal_high)
        last_internal_high := internal_high
    if not na(internal_low)
        last_internal_low := internal_low

var int market_bias = 0
bool bos_bull_raw = confirmed and not na(structure_swing_high) and close > structure_swing_high and (close - open) > safe_atr * bos_body_atr
bool bos_bear_raw = confirmed and not na(structure_swing_low) and close < structure_swing_low and (open - close) > safe_atr * bos_body_atr
bool choch_bull_raw = confirmed and not na(last_internal_high) and close > last_internal_high and (close - open) > safe_atr * choch_body_atr
bool choch_bear_raw = confirmed and not na(last_internal_low) and close < last_internal_low and (open - close) > safe_atr * choch_body_atr

bool bos_bull = bos_bull_raw
bool bos_bear = bos_bear_raw
bool choch_bull = choch_bull_raw and market_bias <= 0
bool choch_bear = choch_bear_raw and market_bias >= 0

if bos_bull or choch_bull
    market_bias := 1
if bos_bear or choch_bear
    market_bias := -1

var int setup_id_bull = 0
var int setup_id_bear = 0
var int consumed_setup_bull = -1
var int consumed_setup_bear = -1
var int consumed_hyper_bull = -1
var int consumed_hyper_bear = -1

if bos_bull or choch_bull
    setup_id_bull += 1
if bos_bear or choch_bear
    setup_id_bear += 1

var int last_sweep_low_bar = na
var int last_sweep_high_bar = na
var int last_bos_bull_bar = na
var int last_bos_bear_bar = na

// =============================================================================
// LIQUIDITY HIERARCHY
// =============================================================================
float sweep_ref_high = na
float sweep_ref_low = na
int sweep_high_idx = -1
int sweep_low_idx = -1

int swing_high_count = array.size(swing_highs)
int swing_high_max = math.max(math.min(swing_high_count - 1, 4), 0)
for i = 0 to swing_high_max
    if i < swing_high_count
        float lvl = array.get(swing_highs, i)
        int lvl_bar = array.get(swing_high_bars, i)
        bool usable = bar_index > lvl_bar and bar_index - lvl_bar <= liq_pivot_lookback
        bool swept = usable and high > lvl and close < lvl
        if swept and na(sweep_ref_high)
            sweep_ref_high := lvl
            sweep_high_idx := i

int swing_low_count = array.size(swing_lows)
int swing_low_max = math.max(math.min(swing_low_count - 1, 4), 0)
for i = 0 to swing_low_max
    if i < swing_low_count
        float lvl = array.get(swing_lows, i)
        int lvl_bar = array.get(swing_low_bars, i)
        bool usable = bar_index > lvl_bar and bar_index - lvl_bar <= liq_pivot_lookback
        bool swept = usable and low < lvl and close > lvl
        if swept and na(sweep_ref_low)
            sweep_ref_low := lvl
            sweep_low_idx := i

float sweep_high_depth = not na(sweep_ref_high) ? (high - sweep_ref_high) / safe_atr : 0.0
float sweep_low_depth = not na(sweep_ref_low) ? (sweep_ref_low - low) / safe_atr : 0.0

bool liq_sweep_high = confirmed and not na(sweep_ref_high) and sweep_high_depth >= sweep_atr_depth_min and bear_rejection >= sweep_reject_min
bool liq_sweep_low = confirmed and not na(sweep_ref_low) and sweep_low_depth >= sweep_atr_depth_min and bull_rejection >= sweep_reject_min

if liq_sweep_low
    last_sweep_low_bar := bar_index
if liq_sweep_high
    last_sweep_high_bar := bar_index
if bos_bull
    last_bos_bull_bar := bar_index
if bos_bear
    last_bos_bear_bar := bar_index

bool eq_high = false
bool eq_low = false
if confirmed and safe_atr > 0.0 and array.size(swing_highs) >= 2
    float tol = safe_atr * eq_atr_tol
    float ph0 = array.get(swing_highs, 0)
    float ph1 = array.get(swing_highs, 1)
    if not na(ph0) and not na(ph1) and math.abs(ph0 - ph1) <= tol
        eq_high := true
if confirmed and safe_atr > 0.0 and array.size(swing_lows) >= 2
    float tol = safe_atr * eq_atr_tol
    float pl0 = array.get(swing_lows, 0)
    float pl1 = array.get(swing_lows, 1)
    if not na(pl0) and not na(pl1) and math.abs(pl0 - pl1) <= tol
        eq_low := true

int liq_weight_high = sweep_high_idx == 0 ? 20 : sweep_high_idx == 1 ? 16 : sweep_high_idx == 2 ? 12 : sweep_high_idx == 3 ? 8 : sweep_high_idx == 4 ? 5 : 0
int liq_weight_low = sweep_low_idx == 0 ? 20 : sweep_low_idx == 1 ? 16 : sweep_low_idx == 2 ? 12 : sweep_low_idx == 3 ? 8 : sweep_low_idx == 4 ? 5 : 0

// =============================================================================
// INDUCEMENT (IDM)
// =============================================================================
var float idm_bull_low = na
var float idm_bear_high = na
var bool idm_bull_active = false
var bool idm_bear_active = false
var int idm_bull_bar = na
var int idm_bear_bar = na

if confirmed and not na(internal_low) and market_bias == 1
    idm_bull_low := internal_low
    idm_bull_active := true
    idm_bull_bar := bar_index
if confirmed and not na(internal_high) and market_bias == -1
    idm_bear_high := internal_high
    idm_bear_active := true
    idm_bear_bar := bar_index

bool idm_bull_ok = not hyper_require_idm or (idm_bull_active and not na(idm_bull_low) and low <= idm_bull_low and close > idm_bull_low)
bool idm_bear_ok = not hyper_require_idm or (idm_bear_active and not na(idm_bear_high) and high >= idm_bear_high and close < idm_bear_high)

if idm_bull_ok
    idm_bull_active := false
if idm_bear_ok
    idm_bear_active := false
if idm_bull_active and bar_index - idm_bull_bar > memory_max_age
    idm_bull_active := false
if idm_bear_active and bar_index - idm_bear_bar > memory_max_age
    idm_bear_active := false

// =============================================================================
// AMD / POWER OF 3
// =============================================================================
var float asia_high = na
var float asia_low = na
var bool asia_set = false

if in_session_asia and not asia_set[1]
    asia_high := high
    asia_low := low
    asia_set := true
else if in_session_asia and asia_set
    asia_high := math.max(asia_high, high)
    asia_low := math.min(asia_low, low)
else if not in_session_asia
    asia_set := false

bool london_sweep_bull = use_killzone_logic and in_session_london and not na(asia_low) and low < asia_low and close > asia_low
bool london_sweep_bear = use_killzone_logic and in_session_london and not na(asia_high) and high > asia_high and close < asia_high
bool ny_expansion_bull = use_killzone_logic and in_session_ny and london_sweep_bull[1]
bool ny_expansion_bear = use_killzone_logic and in_session_ny and london_sweep_bear[1]
// =============================================================================
// OB & FVG
// =============================================================================
var float[] ob_bull_tops = array.new_float(0)
var float[] ob_bull_bots = array.new_float(0)
var int[] ob_bull_scores = array.new_int(0)
var int[] ob_bull_bars = array.new_int(0)
var bool[] ob_bull_valids = array.new_bool(0)

var float[] ob_bear_tops = array.new_float(0)
var float[] ob_bear_bots = array.new_float(0)
var int[] ob_bear_scores = array.new_int(0)
var int[] ob_bear_bars = array.new_int(0)
var bool[] ob_bear_valids = array.new_bool(0)

bool bull_displace = confirmed and close > open and (close - open) > ob_displacement * safe_atr
bool bear_displace = confirmed and close < open and (open - close) > ob_displacement * safe_atr

int ob_bull_lookback = 0
int ob_bear_lookback = 0
if bull_displace
    for j = 1 to 5
        if close[j] < open[j] and ob_bull_lookback == 0
            ob_bull_lookback := j
if bear_displace
    for j = 1 to 5
        if close[j] > open[j] and ob_bear_lookback == 0
            ob_bear_lookback := j

bool ob_bull_event = ob_bull_lookback > 0 and bull_displace
bool ob_bear_event = ob_bear_lookback > 0 and bear_displace

float prev_upper_wick = ob_bull_lookback > 0 ? high[ob_bull_lookback] - math.max(open[ob_bull_lookback], close[ob_bull_lookback]) : 0.0
float prev_lower_wick = ob_bear_lookback > 0 ? math.min(open[ob_bear_lookback], close[ob_bear_lookback]) - low[ob_bear_lookback] : 0.0
float prev_body_bull = ob_bull_lookback > 0 ? math.abs(close[ob_bull_lookback] - open[ob_bull_lookback]) : 0.0
float prev_body_bear = ob_bear_lookback > 0 ? math.abs(close[ob_bear_lookback] - open[ob_bear_lookback]) : 0.0
float prev_rejection_bull = prev_body_bull > 0 ? prev_upper_wick / prev_body_bull : 0.0
float prev_rejection_bear = prev_body_bear > 0 ? prev_lower_wick / prev_body_bear : 0.0
float disp_ratio = safe_div(close - open, safe_atr, 0.0)

var float range_high = na
var float range_low = na
var bool fib_confirmed = false

if confirmed and (bos_bull or bos_bear)
    range_high := nearest_swing_high
    range_low := nearest_swing_low
    fib_confirmed := not na(range_high) and not na(range_low) and range_high > range_low
if confirmed and (choch_bull or choch_bear)
    fib_confirmed := false
    range_high := na
    range_low := na

bool valid_range = not na(range_high) and not na(range_low) and range_high > range_low
float range_size = valid_range ? range_high - range_low : na
float equilibrium = valid_range ? range_low + range_size * 0.5 : na

float bull_ote_top = valid_range ? range_high - range_size * fib_zone_start : na
float bull_ote_bottom = valid_range ? range_high - range_size * fib_zone_end : na
float bear_ote_top = valid_range ? range_low + range_size * fib_zone_end : na
float bear_ote_bottom = valid_range ? range_low + range_size * fib_zone_start : na

bool in_bull_ote = valid_range and low <= bull_ote_top and high >= bull_ote_bottom
bool in_bear_ote = valid_range and high >= bear_ote_bottom and low <= bear_ote_top
bool in_discount = valid_range and close < equilibrium
bool in_premium = valid_range and close > equilibrium

float ob_bull_disp_score = math.min(math.abs(disp_ratio) / math.max(ob_displacement, 0.1), 3.0) * 20.0
float ob_bear_disp_score = math.min(math.abs(disp_ratio) / math.max(ob_displacement, 0.1), 3.0) * 20.0
float ob_vol_score = math.min(vol_ratio, 2.0) * 15.0
float ob_struct_bull_score = bos_bull ? 25.0 : choch_bull ? 15.0 : 0.0
float ob_struct_bear_score = bos_bear ? 25.0 : choch_bear ? 15.0 : 0.0
float ob_loc_bull_score = in_discount ? 20.0 : in_bull_ote ? 15.0 : 0.0
float ob_loc_bear_score = in_premium ? 20.0 : in_bear_ote ? 15.0 : 0.0

float ob_bull_quality_raw = ob_bull_event ? ob_bull_disp_score + ob_vol_score + ob_struct_bull_score + ob_loc_bull_score + (prev_rejection_bull > 0.3 ? 15.0 : 0.0) : 0.0
float ob_bear_quality_raw = ob_bear_event ? ob_bear_disp_score + ob_vol_score + ob_struct_bear_score + ob_loc_bear_score + (prev_rejection_bear > 0.3 ? 15.0 : 0.0) : 0.0
int ob_bull_quality = int(math.round(math.min(math.max(ob_bull_quality_raw, 0.0), 100.0)))
int ob_bear_quality = int(math.round(math.min(math.max(ob_bear_quality_raw, 0.0), 100.0)))

if ob_bull_event and ob_bull_quality >= ob_quality_min
    float bull_top = ob_use_body ? open[ob_bull_lookback] : high[ob_bull_lookback]
    float bull_bot = low[ob_bull_lookback]
    array.unshift(ob_bull_tops, bull_top)
    array.unshift(ob_bull_bots, bull_bot)
    array.unshift(ob_bull_scores, ob_bull_quality)
    array.unshift(ob_bull_bars, bar_index)
    array.unshift(ob_bull_valids, true)
    if array.size(ob_bull_tops) > max_ob_zones
        array.pop(ob_bull_tops)
        array.pop(ob_bull_bots)
        array.pop(ob_bull_scores)
        array.pop(ob_bull_bars)
        array.pop(ob_bull_valids)

if ob_bear_event and ob_bear_quality >= ob_quality_min
    float bear_top = high[ob_bear_lookback]
    float bear_bot = ob_use_body ? open[ob_bear_lookback] : low[ob_bear_lookback]
    array.unshift(ob_bear_tops, bear_top)
    array.unshift(ob_bear_bots, bear_bot)
    array.unshift(ob_bear_scores, ob_bear_quality)
    array.unshift(ob_bear_bars, bar_index)
    array.unshift(ob_bear_valids, true)
    if array.size(ob_bear_tops) > max_ob_zones
        array.pop(ob_bear_tops)
        array.pop(ob_bear_bots)
        array.pop(ob_bear_scores)
        array.pop(ob_bear_bars)
        array.pop(ob_bear_valids)

int ob_bull_count = array.size(ob_bull_tops)
int ob_bear_count = array.size(ob_bear_tops)
int ob_process_limit = math.min(max_ob_zones, 5)

if ob_mitigation
    for ob_i = 0 to ob_process_limit - 1
        if ob_i < ob_bull_count
            if ob_mitigation_mode == "Close" and close < array.get(ob_bull_bots, ob_i)
                array.set(ob_bull_valids, ob_i, false)
            else if ob_mitigation_mode == "Touch" and low <= array.get(ob_bull_bots, ob_i)
                array.set(ob_bull_valids, ob_i, false)
        if ob_i < ob_bear_count
            if ob_mitigation_mode == "Close" and close > array.get(ob_bear_tops, ob_i)
                array.set(ob_bear_valids, ob_i, false)
            else if ob_mitigation_mode == "Touch" and high >= array.get(ob_bear_tops, ob_i)
                array.set(ob_bear_valids, ob_i, false)

if ob_bull_count > 0 or ob_bear_count > 0
    for ob_i = 0 to ob_process_limit - 1
        if ob_i < ob_bull_count
            if bar_index - array.get(ob_bull_bars, ob_i) > memory_max_age
                array.set(ob_bull_valids, ob_i, false)
        if ob_i < ob_bear_count
            if bar_index - array.get(ob_bear_bars, ob_i) > memory_max_age
                array.set(ob_bear_valids, ob_i, false)

bool any_bull_ob_valid = false
bool any_bear_ob_valid = false
float best_bull_ob_bot = na
float best_bear_ob_top = na
int best_bull_ob_score = 0
int best_bear_ob_score = 0

if ob_bull_count > 0 or ob_bear_count > 0
    for ob_i = 0 to ob_process_limit - 1
        if ob_i < ob_bull_count
            if array.get(ob_bull_valids, ob_i)
                any_bull_ob_valid := true
                if na(best_bull_ob_bot) or array.get(ob_bull_bots, ob_i) > best_bull_ob_bot
                    best_bull_ob_bot := array.get(ob_bull_bots, ob_i)
                    best_bull_ob_score := array.get(ob_bull_scores, ob_i)
        if ob_i < ob_bear_count
            if array.get(ob_bear_valids, ob_i)
                any_bear_ob_valid := true
                if na(best_bear_ob_top) or array.get(ob_bear_tops, ob_i) < best_bear_ob_top
                    best_bear_ob_top := array.get(ob_bear_tops, ob_i)
                    best_bear_ob_score := array.get(ob_bear_scores, ob_i)

// FVG + Inversion
var float[] fvg_bull_tops = array.new_float(0)
var float[] fvg_bull_bots = array.new_float(0)
var int[] fvg_bull_bars = array.new_int(0)
var bool[] fvg_bull_valids = array.new_bool(0)
var float[] fvg_bear_tops = array.new_float(0)
var float[] fvg_bear_bots = array.new_float(0)
var int[] fvg_bear_bars = array.new_int(0)
var bool[] fvg_bear_valids = array.new_bool(0)

var float[] inv_bull_tops = array.new_float(0)
var float[] inv_bull_bots = array.new_float(0)
var int[] inv_bull_bars = array.new_int(0)
var bool[] inv_bull_valids = array.new_bool(0)
var float[] inv_bear_tops = array.new_float(0)
var float[] inv_bear_bots = array.new_float(0)
var int[] inv_bear_bars = array.new_int(0)
var bool[] inv_bear_valids = array.new_bool(0)

bool middle_bull_disp = close[1] > open[1] and math.abs(close[1] - open[1]) >= safe_atr[1] * fvg_middle_body_atr
bool middle_bear_disp = close[1] < open[1] and math.abs(open[1] - close[1]) >= safe_atr[1] * fvg_middle_body_atr
bool fvg_bull_raw = bar_index > 2 and low > high[2]
bool fvg_bear_raw = bar_index > 2 and high < low[2]
float fvg_bull_gap = fvg_bull_raw ? low - high[2] : 0.0
float fvg_bear_gap = fvg_bear_raw ? low[2] - high : 0.0
bool fvg_bull_event = confirmed and fvg_bull_raw and fvg_bull_gap >= fvg_min_size * safe_atr and (not fvg_require_middle_displacement or middle_bull_disp)
bool fvg_bear_event = confirmed and fvg_bear_raw and fvg_bear_gap >= fvg_min_size * safe_atr and (not fvg_require_middle_displacement or middle_bear_disp)

if fvg_bull_event
    array.unshift(fvg_bull_tops, low)
    array.unshift(fvg_bull_bots, high[2])
    array.unshift(fvg_bull_bars, bar_index)
    array.unshift(fvg_bull_valids, true)
    if array.size(fvg_bull_tops) > 5
        array.pop(fvg_bull_tops)
        array.pop(fvg_bull_bots)
        array.pop(fvg_bull_bars)
        array.pop(fvg_bull_valids)

if fvg_bear_event
    array.unshift(fvg_bear_tops, low[2])
    array.unshift(fvg_bear_bots, high)
    array.unshift(fvg_bear_bars, bar_index)
    array.unshift(fvg_bear_valids, true)
    if array.size(fvg_bear_tops) > 5
        array.pop(fvg_bear_tops)
        array.pop(fvg_bear_bots)
        array.pop(fvg_bear_bars)
        array.pop(fvg_bear_valids)

int fvg_bull_count = array.size(fvg_bull_tops)
int fvg_bear_count = array.size(fvg_bear_tops)
int fvg_process_limit = 5

for fvg_i = 0 to fvg_process_limit - 1
    if fvg_i < fvg_bull_count
        bool mitigated = fvg_mitigation_mode == "Close" ? close <= array.get(fvg_bull_bots, fvg_i) : low <= array.get(fvg_bull_bots, fvg_i)
        if mitigated and array.get(fvg_bull_valids, fvg_i)
            array.set(fvg_bull_valids, fvg_i, false)
            if use_inversion_fvg
                array.unshift(inv_bull_tops, array.get(fvg_bull_tops, fvg_i))
                array.unshift(inv_bull_bots, array.get(fvg_bull_bots, fvg_i))
                array.unshift(inv_bull_bars, bar_index)
                array.unshift(inv_bull_valids, true)
                if array.size(inv_bull_tops) > 3
                    array.pop(inv_bull_tops)
                    array.pop(inv_bull_bots)
                    array.pop(inv_bull_bars)
                    array.pop(inv_bull_valids)
    if fvg_i < fvg_bear_count
        bool mitigated = fvg_mitigation_mode == "Close" ? close >= array.get(fvg_bear_tops, fvg_i) : high >= array.get(fvg_bear_tops, fvg_i)
        if mitigated and array.get(fvg_bear_valids, fvg_i)
            array.set(fvg_bear_valids, fvg_i, false)
            if use_inversion_fvg
                array.unshift(inv_bear_tops, array.get(fvg_bear_tops, fvg_i))
                array.unshift(inv_bear_bots, array.get(fvg_bear_bots, fvg_i))
                array.unshift(inv_bear_bars, bar_index)
                array.unshift(inv_bear_valids, true)
                if array.size(inv_bear_tops) > 3
                    array.pop(inv_bear_tops)
                    array.pop(inv_bear_bots)
                    array.pop(inv_bear_bars)
                    array.pop(inv_bear_valids)

if fvg_bull_count > 0 or fvg_bear_count > 0
    for fvg_i = 0 to fvg_process_limit - 1
        if fvg_i < fvg_bull_count
            if bar_index - array.get(fvg_bull_bars, fvg_i) > memory_max_age
                array.set(fvg_bull_valids, fvg_i, false)
        if fvg_i < fvg_bear_count
            if bar_index - array.get(fvg_bear_bars, fvg_i) > memory_max_age
                array.set(fvg_bear_valids, fvg_i, false)

bool any_bull_fvg_valid = false
bool any_bear_fvg_valid = false
float best_bull_fvg_bot = na
float best_bear_fvg_top = na

if fvg_bull_count > 0 or fvg_bear_count > 0
    for fvg_i = 0 to fvg_process_limit - 1
        if fvg_i < fvg_bull_count
            if array.get(fvg_bull_valids, fvg_i)
                any_bull_fvg_valid := true
                if na(best_bull_fvg_bot) or array.get(fvg_bull_bots, fvg_i) > best_bull_fvg_bot
                    best_bull_fvg_bot := array.get(fvg_bull_bots, fvg_i)
        if fvg_i < fvg_bear_count
            if array.get(fvg_bear_valids, fvg_i)
                any_bear_fvg_valid := true
                if na(best_bear_fvg_top) or array.get(fvg_bear_tops, fvg_i) < best_bear_fvg_top
                    best_bear_fvg_top := array.get(fvg_bear_tops, fvg_i)

// =============================================================================
// ROLLING VOLUME NODE
// =============================================================================
var float[] rvn_prices = array.new_float(0)
var float[] rvn_volumes = array.new_float(0)
var int[] rvn_bars = array.new_int(0)

if confirmed and use_vol_profile
    float mid_price = (high + low) / 2.0
    float bin_width = safe_atr * 0.5
    int rvn_size = array.size(rvn_prices)
    int closest_idx = -1
    float closest_dist = 999999.0
    int check_limit = math.min(rvn_size - 1, vp_bins - 1)
    if rvn_size > 0
        for k = 0 to check_limit
            float dist = math.abs(array.get(rvn_prices, k) - mid_price)
            if dist < closest_dist
                closest_dist := dist
                closest_idx := k
    if closest_idx >= 0 and closest_dist <= bin_width
        array.set(rvn_volumes, closest_idx, array.get(rvn_volumes, closest_idx) + volume)
    else if rvn_size < vp_bins
        array.push(rvn_prices, mid_price)
        array.push(rvn_volumes, volume)
        array.push(rvn_bars, bar_index)
    int rvn_current_size = array.size(rvn_bars)
    if rvn_current_size > 0
        int oldest_idx = 0
        bool removed = false
        for rvn_i = 0 to rvn_current_size - 1
            if not removed and bar_index - array.get(rvn_bars, rvn_i) > vp_lookback
                oldest_idx := rvn_i
                removed := true
        if removed
            array.remove(rvn_prices, oldest_idx)
            array.remove(rvn_volumes, oldest_idx)
            array.remove(rvn_bars, oldest_idx)

float rvn_price = na
float rvn_vol = 0.0
int rvn_size_now = array.size(rvn_prices)
int rvn_limit = math.min(rvn_size_now - 1, vp_bins - 1)
if use_vol_profile and rvn_size_now > 0
    for k = 0 to rvn_limit
        float v = array.get(rvn_volumes, k)
        if v > rvn_vol
            rvn_vol := v
            rvn_price := array.get(rvn_prices, k)

bool near_poc = use_vol_profile and not na(rvn_price) and math.abs(close - rvn_price) <= safe_atr * 0.3

// =============================================================================
// REGIME / MOMENTUM / PHYSICS
// =============================================================================
[diplus, diminus, adx_raw] = ta.dmi(adx_len, adx_len)
float adx_val = adx_raw
bool dmi_bull = diplus > diminus
bool dmi_bear = diminus > diplus
bool adx_trend = use_regime ? adx_val >= adx_trend_thresh : true
bool adx_range = use_regime ? adx_val <= adx_range_thresh : false

float atr_rank = ta.percentrank(atr_val, atr_percentile_len)
bool atr_expanding = not use_volatility_regime or atr_rank >= atr_expansion_rank
bool atr_extreme = use_volatility_regime and atr_rank >= atr_extreme_rank
bool regime_trade_ok = (not use_regime or adx_trend) and not atr_extreme
bool regime_range_ok = use_regime and adx_range
bool regime_trend = adx_trend and not atr_extreme
bool regime_expansion = atr_expanding and not adx_range
bool regime_compression = atr_rank < 30.0 and not adx_trend

float rsi_val = ta.rsi(close, rsi_len)
float body_mom = math.abs(close - open)
float stable_body = math.max(body_mom, safe_atr * 0.2)
float wick_mom = math.max((high - low) - stable_body, 0.0)
float candle_eff = stable_body > 0 ? wick_mom / stable_body : 0.0
int candle_direction = close > open ? 1 : close < open ? -1 : 0
float range_norm = candle_range / safe_atr
float vf_raw = range_norm * vol_ratio / (candle_eff + 1.0 + 0.000001) * candle_direction
float vf_ema1 = ta.ema(vf_raw, mom_lookback)
float vf_ema2 = ta.ema(vf_ema1, mom_lookback)
float vf_dema = 2.0 * vf_ema1 - vf_ema2
float atr_sma_50 = ta.sma(atr_val, 50)
int norm_period = not na(atr_sma_50) and atr_val > atr_sma_50 ? 30 : 60
float vf_rank = ta.percentrank(vf_dema, norm_period)

bool rsi_momentum_bull = rsi_val > rsi_bull_thresh and vol_ok
bool rsi_momentum_bear = rsi_val < rsi_bear_thresh and vol_ok
bool original_momentum_bull = vf_rank > momentum_threshold and vol_ok
bool original_momentum_bear = vf_rank < (100.0 - momentum_threshold) and vol_ok

bool momentum_bull = switch momentum_method
    "RSI" => rsi_momentum_bull
    "Original" => original_momentum_bull
    => rsi_momentum_bull or original_momentum_bull
bool momentum_bear = switch momentum_method
    "RSI" => rsi_momentum_bear
    "Original" => original_momentum_bear
    => rsi_momentum_bear or original_momentum_bear

float rsi_strength = math.abs(rsi_val - 50.0) / 50.0
float original_strength = math.abs(vf_rank - 50.0) / 50.0
float momentum_strength = math.min(math.max(math.max(rsi_strength, original_strength), 0.0), 1.0)

float velocity = safe_div(close - close[physics_len], safe_atr * physics_len, 0.0)
float acceleration = velocity - nz(velocity[1], velocity)
float path_distance = math.sum(math.abs(ta.change(close)), physics_len)
float efficiency = path_distance > 0 ? math.abs(close - close[physics_len]) / path_distance : 0.0
float signed_pressure_raw = candle_direction * vol_ratio * safe_div(body, safe_atr, 0.0)
int physics_smooth_len = math.max(2, int(math.round(physics_len / 2.0)))
float signed_pressure = ta.ema(signed_pressure_raw, physics_smooth_len)

bool physics_bull = velocity > velocity_min and acceleration >= -velocity_min and efficiency >= efficiency_min and signed_pressure > pressure_min
bool physics_bear = velocity < -velocity_min and acceleration <= velocity_min and efficiency >= efficiency_min and signed_pressure < -pressure_min

float avg_displacement = ta.sma(math.abs(close - open), 10)
float void_ratio = safe_div(math.abs(close - open), avg_displacement, 0.0)
bool liquidity_void_bull = confirmed and close > open and void_ratio > 2.0 and candle_eff < 0.3
bool liquidity_void_bear = confirmed and close < open and void_ratio > 2.0 and candle_eff < 0.3

// =============================================================================
// V13 STEP 4: ADVANCED REGIME MATRIX
// =============================================================================
float rm_low_vol_rank = 20.0
float rm_compression_rank = 40.0
float rm_news_vol_ratio = 1.80
float rm_news_range_atr = 2.50

bool rm_low_vol = atr_rank <= rm_low_vol_rank
bool rm_compression_vol = atr_rank <= rm_compression_rank
bool rm_high_vol = atr_rank >= atr_extreme_rank
bool rm_expansion_vol = atr_rank >= atr_expansion_rank

bool rm_news_impulse = confirmed and rm_high_vol and void_ratio > 2.0 and vol_ratio >= rm_news_vol_ratio and range_norm >= rm_news_range_atr
bool rm_liq_void = confirmed and (liquidity_void_bull or liquidity_void_bear)

int regime_matrix = 0
string regime_matrix_text = "UNKNOWN"
color regime_matrix_color = color.gray
float regime_matrix_quality = 0.50

if rm_news_impulse
    regime_matrix := 8
    regime_matrix_text := "NEWS IMPULSE"
    regime_matrix_color := #FF5252
    regime_matrix_quality := 0.10
else if rm_liq_void
    regime_matrix := 7
    regime_matrix_text := "LIQUIDITY VOID"
    regime_matrix_color := #9C27B0
    regime_matrix_quality := 0.55
else if rm_high_vol
    regime_matrix := 5
    regime_matrix_text := "HIGH VOLATILITY"
    regime_matrix_color := #EF5350
    regime_matrix_quality := 0.25
else if rm_low_vol
    regime_matrix := 6
    regime_matrix_text := "LOW VOLATILITY"
    regime_matrix_color := #607D8B
    regime_matrix_quality := 0.35
else if adx_trend and rm_expansion_vol
    regime_matrix := 1
    regime_matrix_text := "TREND EXPANSION"
    regime_matrix_color := #26A69A
    regime_matrix_quality := 1.00
else if adx_trend and rm_compression_vol
    regime_matrix := 2
    regime_matrix_text := "TREND COMPRESSION"
    regime_matrix_color := #4CAF50
    regime_matrix_quality := 0.80
else if adx_range and rm_expansion_vol
    regime_matrix := 3
    regime_matrix_text := "RANGE EXPANSION"
    regime_matrix_color := #FFB300
    regime_matrix_quality := 0.40
else if adx_range and rm_compression_vol
    regime_matrix := 4
    regime_matrix_text := "RANGE COMPRESSION"
    regime_matrix_color := #F2C94C
    regime_matrix_quality := 0.30
else if adx_trend
    regime_matrix := 1
    regime_matrix_text := "TREND EXPANSION"
    regime_matrix_color := #26A69A
    regime_matrix_quality := 0.90
else if adx_range
    regime_matrix := 4
    regime_matrix_text := "RANGE COMPRESSION"
    regime_matrix_color := #F2C94C
    regime_matrix_quality := 0.30
else
    regime_matrix := 0
    regime_matrix_text := "UNKNOWN"
    regime_matrix_color := color.gray
    regime_matrix_quality := 0.50

bool regime_matrix_trade_ok = regime_matrix_quality >= 0.55
bool regime_matrix_aggressive_ok = regime_matrix_quality >= 0.35

// =============================================================================
// MTF DATA REQUESTS (moved from L16 for early availability)
// =============================================================================
[mtf1_close, mtf1_ema] = request.security(syminfo.tickerid, mtf_tf1, [close, ta.ema(close, 50)], lookahead=barmerge.lookahead_off)
[mtf2_close, mtf2_ema] = request.security(syminfo.tickerid, mtf_tf2, [close, ta.ema(close, 50)], lookahead=barmerge.lookahead_off)
[mtf3_close, mtf3_ema] = request.security(syminfo.tickerid, mtf_tf3, [close, ta.ema(close, 50)], lookahead=barmerge.lookahead_off)
[mtf_macro_close, mtf_macro_ema] = request.security(syminfo.tickerid, mtf_macro_tf, [close, ta.ema(close, 50)], lookahead=barmerge.lookahead_off)

string mtf1_state = not na(mtf1_close) and not na(mtf1_ema) ? (mtf1_close > mtf1_ema ? "صاعد" : "هابط") : "N/A"
string mtf2_state = not na(mtf2_close) and not na(mtf2_ema) ? (mtf2_close > mtf2_ema ? "صاعد" : "هابط") : "N/A"
string mtf3_state = not na(mtf3_close) and not na(mtf3_ema) ? (mtf3_close > mtf3_ema ? "صاعد" : "هابط") : "N/A"

// =============================================================================
// V13 STEP 6: MTF ARCHITECTURE ENGINE
// =============================================================================
int mtf_macro_bias = not na(mtf_macro_close) and not na(mtf_macro_ema) ? (mtf_macro_close > mtf_macro_ema ? 1 : mtf_macro_close < mtf_macro_ema ? -1 : 0) : 0
int mtf_htf_bias = not na(htf_close) and not na(htf_ema) ? (htf_close > htf_ema ? 1 : htf_close < htf_ema ? -1 : 0) : 0
int mtf2_bias = not na(mtf2_close) and not na(mtf2_ema) ? (mtf2_close > mtf2_ema ? 1 : mtf2_close < mtf2_ema ? -1 : 0) : 0
int mtf1_bias = not na(mtf1_close) and not na(mtf1_ema) ? (mtf1_close > mtf1_ema ? 1 : mtf1_close < mtf1_ema ? -1 : 0) : 0

int mtf_bull_alignment = (mtf_macro_bias == 1 ? 1 : 0) + (mtf_htf_bias == 1 ? 1 : 0) + (mtf2_bias == 1 ? 1 : 0) + (mtf1_bias == 1 ? 1 : 0)
int mtf_bear_alignment = (mtf_macro_bias == -1 ? 1 : 0) + (mtf_htf_bias == -1 ? 1 : 0) + (mtf2_bias == -1 ? 1 : 0) + (mtf1_bias == -1 ? 1 : 0)

float mtf_alignment_score_bull = mtf_bull_alignment / 4.0 * 100.0
float mtf_alignment_score_bear = mtf_bear_alignment / 4.0 * 100.0

bool mtf_bull_aligned = mtf_bull_alignment >= mtf_alignment_min
bool mtf_bear_aligned = mtf_bear_alignment >= mtf_alignment_min

int mtf_bonus_bull = use_mtf_architecture ? int(math.round(mtf_alignment_score_bull * 0.05)) : 0
int mtf_bonus_bear = use_mtf_architecture ? int(math.round(mtf_alignment_score_bear * 0.05)) : 0

bool mtf_strong_bull_ok = not use_mtf_architecture or not mtf_require_alignment or mtf_bull_aligned
bool mtf_strong_bear_ok = not use_mtf_architecture or not mtf_require_alignment or mtf_bear_aligned

// =============================================================================
// V13 STEP 1: EVIDENCE SCORING — CLUSTER-AWARE ENGINE
// =============================================================================
bool trend_profile = adaptive_profile == "ذهب/بتكوين ترند"
bool forex_profile = adaptive_profile == "فوركس سيولة"
int profile_trend_bonus = use_adaptive_profile and trend_profile ? 5 : 0
int profile_liq_bonus = use_adaptive_profile and forex_profile ? 5 : 0

int _sweep_bull_age = ta.barssince(liq_sweep_low)
int _sweep_bear_age = ta.barssince(liq_sweep_high)
int _bos_bull_age = ta.barssince(bos_bull)
int _bos_bear_age = ta.barssince(bos_bear)
int _choch_bull_age = ta.barssince(choch_bull)
int _choch_bear_age = ta.barssince(choch_bear)
int _ob_bull_age = ta.barssince(ob_bull_event and ob_bull_quality >= ob_quality_min)
int _ob_bear_age = ta.barssince(ob_bear_event and ob_bear_quality >= ob_quality_min)
int _fvg_bull_age = ta.barssince(fvg_bull_event)
int _fvg_bear_age = ta.barssince(fvg_bear_event)

bool _sweep_ok_bull = age_ok(_sweep_bull_age, memory_max_age)
bool _sweep_ok_bear = age_ok(_sweep_bear_age, memory_max_age)
bool _bos_ok_bull = age_ok(_bos_bull_age, memory_max_age)
bool _bos_ok_bear = age_ok(_bos_bear_age, memory_max_age)
bool _choch_ok_bull = age_ok(_choch_bull_age, memory_max_age)
bool _choch_ok_bear = age_ok(_choch_bear_age, memory_max_age)
bool _ob_ok_bull = age_ok(_ob_bull_age, memory_max_age) and any_bull_ob_valid
bool _ob_ok_bear = age_ok(_ob_bear_age, memory_max_age) and any_bear_ob_valid
bool _fvg_ok_bull = age_ok(_fvg_bull_age, memory_max_age) and any_bull_fvg_valid
bool _fvg_ok_bear = age_ok(_fvg_bear_age, memory_max_age) and any_bear_fvg_valid

// --- CLUSTER 1: STRUCTURE & LIQUIDITY (Max 25) ---
int raw_struct_liq_bull = (_sweep_ok_bull ? 16 + profile_liq_bonus : 0) + (_bos_ok_bull ? 17 : _choch_ok_bull ? 9 : 0)
int raw_struct_liq_bear = (_sweep_ok_bear ? 16 + profile_liq_bonus : 0) + (_bos_ok_bear ? 17 : _choch_ok_bear ? 9 : 0)
int struct_bull_score = math.min(raw_struct_liq_bull, 25)
int struct_bear_score = math.min(raw_struct_liq_bear, 25)

// --- CLUSTER 2: ZONES (Max 20) + Correlation Penalty ---
int raw_zone_bull = (_ob_ok_bull ? 16 : 0) + (_fvg_ok_bull ? 13 : 0) + (near_poc ? 5 : 0)
int raw_zone_bear = (_ob_ok_bear ? 16 : 0) + (_fvg_ok_bear ? 13 : 0) + (near_poc ? 5 : 0)
float zone_penalty_bull = (_ob_ok_bull and _fvg_ok_bull) ? 0.6 : 1.0
float zone_penalty_bear = (_ob_ok_bear and _fvg_ok_bear) ? 0.6 : 1.0
int zone_bull_score = int(math.min(math.round(raw_zone_bull * zone_penalty_bull), 20))
int zone_bear_score = int(math.min(math.round(raw_zone_bear * zone_penalty_bear), 20))

// --- CLUSTER 3: LOCATION (Max 15) ---
int loc_bull_score = math.min((in_bull_ote ? 9 : 0) + (in_discount ? 7 : 0), 15)
int loc_bear_score = math.min((in_bear_ote ? 9 : 0) + (in_premium ? 7 : 0), 15)

// --- CLUSTER 4: DYNAMICS — Momentum + Physics merged (Max 15) ---
int raw_mom_bull = 0
int raw_mom_bear = 0
if momentum_method == "RSI" or momentum_method == "Hybrid"
    raw_mom_bull += rsi_momentum_bull ? 9 : 0
    raw_mom_bear += rsi_momentum_bear ? 9 : 0
if momentum_method == "Original" or momentum_method == "Hybrid"
    raw_mom_bull += original_momentum_bull ? 9 : 0
    raw_mom_bear += original_momentum_bear ? 9 : 0

int raw_phys_bull = physics_bull ? 10 : 0
int raw_phys_bear = physics_bear ? 10 : 0

float cluster4_raw_bull = math.max(raw_mom_bull, raw_phys_bull) + (math.min(raw_mom_bull, raw_phys_bull) * 0.3)
float cluster4_raw_bear = math.max(raw_mom_bear, raw_phys_bear) + (math.min(raw_mom_bear, raw_phys_bear) * 0.3)
int mom_bull_score = int(math.min(math.round(cluster4_raw_bull), 15))
int mom_bear_score = int(math.min(math.round(cluster4_raw_bear), 15))
int phys_bull_score = 0
int phys_bear_score = 0

// --- CLUSTER 5: CONTEXT — HTF + Volatility merged (Max 15) ---
int raw_htf_bull = 0
int raw_htf_bear = 0
if use_htf_trend
    raw_htf_bull += strong_htf_bull_ok ? 5 + profile_trend_bonus : 0
    raw_htf_bear += strong_htf_bear_ok ? 5 + profile_trend_bonus : 0
    raw_htf_bull += htf_power_bull ? 6 : 0
    raw_htf_bear += htf_power_bear ? 6 : 0
int vol_bonus_bull = atr_expanding ? 3 : 0
int vol_bonus_bear = atr_expanding ? 3 : 0
int htf_bull_score = math.min(raw_htf_bull + vol_bonus_bull, 15)
int htf_bear_score = math.min(raw_htf_bear + vol_bonus_bear, 15)
vol_bonus_bull := 0
vol_bonus_bear := 0

// --- RISK (added later in Risk Engine) ---
int risk_bull_score = 0
int risk_bear_score = 0

// --- TOTAL RAW SCORE ---
int bull_score = struct_bull_score + zone_bull_score + loc_bull_score + mom_bull_score + phys_bull_score + htf_bull_score + vol_bonus_bull
int bear_score = struct_bear_score + zone_bear_score + loc_bear_score + mom_bear_score + phys_bear_score + htf_bear_score + vol_bonus_bear
bull_score := math.min(bull_score, 100)
bear_score := math.min(bear_score, 100)

// --- EVIDENCE STRINGS ---
string bull_evidence = ""
string bear_evidence = ""
bull_evidence := bull_evidence + (_sweep_ok_bull ? "Liq+ " : "")
bull_evidence := bull_evidence + (_bos_ok_bull ? "BOS+ " : _choch_ok_bull ? "CH+ " : "")
bull_evidence := bull_evidence + (_ob_ok_bull ? "OB+ " : "")
bull_evidence := bull_evidence + (_fvg_ok_bull ? "FVG+ " : "")
bull_evidence := bull_evidence + (in_bull_ote ? "OTE+ " : "")
bull_evidence := bull_evidence + (in_discount ? "Disc+ " : "")
bull_evidence := bull_evidence + (momentum_bull ? "Mom+ " : "")
bull_evidence := bull_evidence + (physics_bull ? "Phys+ " : "")
bull_evidence := bull_evidence + (htf_power_bull ? "Power+ " : htf_trend_up ? "HTF+ " : "")
bull_evidence := bull_evidence + (near_poc ? "RVN+ " : "")

bear_evidence := bear_evidence + (_sweep_ok_bear ? "Liq+ " : "")
bear_evidence := bear_evidence + (_bos_ok_bear ? "BOS+ " : _choch_ok_bear ? "CH+ " : "")
bear_evidence := bear_evidence + (_ob_ok_bear ? "OB+ " : "")
bear_evidence := bear_evidence + (_fvg_ok_bear ? "FVG+ " : "")
bear_evidence := bear_evidence + (in_bear_ote ? "OTE+ " : "")
bear_evidence := bear_evidence + (in_premium ? "Prem+ " : "")
bear_evidence := bear_evidence + (momentum_bear ? "Mom+ " : "")
bear_evidence := bear_evidence + (physics_bear ? "Phys+ " : "")
bear_evidence := bear_evidence + (htf_power_bear ? "Power+ " : htf_trend_down ? "HTF+ " : "")
bear_evidence := bear_evidence + (near_poc ? "RVN+ " : "")

// =============================================================================
// RISK ENGINE
// =============================================================================
float bull_sl = na
float bull_tp = na
float bear_sl = na
float bear_tp = na
float bull_rr = na
float bear_rr = na
bool bull_rr_ok = not use_risk_filter
bool bear_rr_ok = not use_risk_filter

if any_bull_ob_valid or any_bull_fvg_valid or _sweep_ok_bull
    float structure_stop = any_bull_ob_valid and not na(best_bull_ob_bot) ? best_bull_ob_bot : any_bull_fvg_valid and not na(best_bull_fvg_bot) ? best_bull_fvg_bot : not na(nearest_swing_low) ? nearest_swing_low : low
    bull_sl := structure_stop - safe_atr * sl_atr_buffer
    float bull_risk = math.max(close - bull_sl, safe_atr * 0.05)
    float real_target = na
    if not na(nearest_swing_high) and nearest_swing_high > close
        real_target := nearest_swing_high
    else if eq_high and not na(sweep_ref_high) and sweep_ref_high > close
        real_target := sweep_ref_high
    if not na(real_target)
        bull_tp := real_target
        float bull_reward = math.max(bull_tp - close, safe_atr * 0.05)
        bull_rr := safe_div(bull_reward, bull_risk, 0.0)
        bull_rr_ok := bull_rr >= rr_min
    else
        bull_rr_ok := false

if any_bear_ob_valid or any_bear_fvg_valid or _sweep_ok_bear
    float structure_stop = any_bear_ob_valid and not na(best_bear_ob_top) ? best_bear_ob_top : any_bear_fvg_valid and not na(best_bear_fvg_top) ? best_bear_fvg_top : not na(nearest_swing_high) ? nearest_swing_high : high
    bear_sl := structure_stop + safe_atr * sl_atr_buffer
    float bear_risk = math.max(bear_sl - close, safe_atr * 0.05)
    float real_target = na
    if not na(nearest_swing_low) and nearest_swing_low < close
        real_target := nearest_swing_low
    else if eq_low and not na(sweep_ref_low) and sweep_ref_low < close
        real_target := sweep_ref_low
    if not na(real_target)
        bear_tp := real_target
        float bear_reward = math.max(close - bear_tp, safe_atr * 0.05)
        bear_rr := safe_div(bear_reward, bear_risk, 0.0)
        bear_rr_ok := bear_rr >= rr_min
    else
        bear_rr_ok := false

if use_risk_filter
    risk_bull_score += bull_rr_ok ? 5 : 0
    risk_bear_score += bear_rr_ok ? 5 : 0
    bull_score := math.min(bull_score + risk_bull_score, 100)
    bear_score := math.min(bear_score + risk_bear_score, 100)
    bull_evidence := bull_evidence + (bull_rr_ok ? "RR+ " : "")
    bear_evidence := bear_evidence + (bear_rr_ok ? "RR+ " : "")

// =============================================================================
// V13 STEP 6: MTF ARCHITECTURE BONUS
// =============================================================================
if use_mtf_architecture
    bull_score := math.min(bull_score + mtf_bonus_bull, 100)
    bear_score := math.min(bear_score + mtf_bonus_bear, 100)
    bull_evidence := bull_evidence + (mtf_bull_aligned ? "MTF+ " : "")
    bear_evidence := bear_evidence + (mtf_bear_aligned ? "MTF+ " : "")

// =============================================================================
// V13 STEP 2: DYNAMIC EXECUTION COST MODEL
// =============================================================================
float tick_value = syminfo.mintick
float session_cost_mult = in_session_asia ? 1.30 : (in_session_london or in_session_ny) ? 1.0 : 1.15
float vol_slippage_mult = atr_rank >= 90.0 ? 1.80 : atr_rank >= 75.0 ? 1.40 : atr_rank >= 50.0 ? 1.15 : 1.0
float current_body_atr = body / safe_atr
float displacement_slippage_mult = current_body_atr > 1.5 ? 1.50 : current_body_atr > 1.0 ? 1.20 : 1.0

float dynamic_spread = spread_ticks * session_cost_mult
float dynamic_slippage = slippage_ticks * vol_slippage_mult * displacement_slippage_mult
float dynamic_commission = commission_ticks * 2.0

float total_dynamic_cost_ticks = dynamic_spread + dynamic_commission + dynamic_slippage
float cost_r = safe_div(total_dynamic_cost_ticks * tick_value, safe_atr, 0.0)
cost_r := math.min(cost_r, 0.5)

// =============================================================================
// L13: DECISION ENGINE — SETUP STATE MACHINE
// =============================================================================
int quick_max_age = math.min(3, memory_max_age)
bool sweep_ok_bull_quick = age_ok(_sweep_bull_age, quick_max_age)
bool sweep_ok_bear_quick = age_ok(_sweep_bear_age, quick_max_age)
bool sweep_ok_bull = _sweep_ok_bull
bool sweep_ok_bear = _sweep_ok_bear
bool bos_ok_bull = _bos_ok_bull
bool bos_ok_bear = _bos_ok_bear
bool choch_ok_bull = _choch_ok_bull
bool choch_ok_bear = _choch_ok_bear
bool ob_ok_bull = _ob_ok_bull
bool ob_ok_bear = _ob_ok_bear
bool fvg_ok_bull = _fvg_ok_bull
bool fvg_ok_bear = _fvg_ok_bear

bool sequence_ok_bull = not strong_require_sequence or (not na(last_sweep_low_bar) and not na(last_bos_bull_bar) and last_sweep_low_bar < last_bos_bull_bar and last_bos_bull_bar - last_sweep_low_bar <= 5 and bar_index - last_bos_bull_bar <= memory_max_age)
bool sequence_ok_bear = not strong_require_sequence or (not na(last_sweep_high_bar) and not na(last_bos_bear_bar) and last_sweep_high_bar < last_bos_bear_bar and last_bos_bear_bar - last_sweep_high_bar <= 5 and bar_index - last_bos_bear_bar <= memory_max_age)

bool premium_discount_bull_ok = (not require_discount_for_longs) or in_discount
bool premium_discount_bear_ok = (not require_premium_for_shorts) or in_premium
bool hyper_pd_bull_ok = (not hyper_require_premium_discount) or in_discount
bool hyper_pd_bear_ok = (not hyper_require_premium_discount) or in_premium

// =============================================================================
// L14: SIGNAL EXECUTION (with V13 MTF Filter)
// =============================================================================
float roc_short = ta.roc(close, 5)
bool fast_momentum_up = confirmed and roc_short > 0.20 and vol_ok
bool fast_momentum_down = confirmed and roc_short < -0.20 and vol_ok
bool confirmed_market_ok = (not no_trade_when_range or not regime_range_ok) and (not no_trade_when_extreme_atr or not atr_extreme)

bool hard_block_short = use_hard_trend_guard and (htf_power_bull or (use_htf_trend and htf_trend_up and trend_profile))
bool hard_block_long = use_hard_trend_guard and htf_power_bear and adaptive_profile == "قناص صارم"

bool quick_base_bull = confirmed and confirmed_market_ok and not hard_block_long and fast_momentum_up and (not quick_require_trend or close > sma20) and bull_rr_ok and session_ok and (not quick_require_sweep or sweep_ok_bull_quick) and bull_score >= profile_quick_score_min
bool quick_base_bear = confirmed and confirmed_market_ok and not hard_block_short and fast_momentum_down and (not quick_require_trend or close < sma20) and bear_rr_ok and session_ok and (not quick_require_sweep or sweep_ok_bear_quick) and bear_score >= profile_quick_score_min

bool strong_base_bull = confirmed and confirmed_market_ok and not hard_block_long and bull_score >= profile_strong_score_min and bos_ok_bull and (ob_ok_bull or fvg_ok_bull) and sequence_ok_bull and (not strong_require_fib or in_bull_ote) and strong_session_ok and strong_htf_bull_ok and premium_discount_bull_ok and bull_rr_ok and regime_trade_ok and mtf_strong_bull_ok
bool strong_base_bear = confirmed and confirmed_market_ok and not hard_block_short and bear_score >= profile_strong_score_min and bos_ok_bear and (ob_ok_bear or fvg_ok_bear) and sequence_ok_bear and (not strong_require_fib or in_bear_ote) and strong_session_ok and strong_htf_bear_ok and premium_discount_bear_ok and bear_rr_ok and regime_trade_ok and mtf_strong_bear_ok

bool hyper_base_bull = confirmed and strong_base_bull and bull_score >= profile_hyper_score_min and momentum_bull and momentum_strength > 0.55 and (not hyper_require_physics or physics_bull) and hyper_pd_bull_ok and not atr_extreme and idm_bull_ok
bool hyper_base_bear = confirmed and strong_base_bear and bear_score >= profile_hyper_score_min and momentum_bear and momentum_strength > 0.55 and (not hyper_require_physics or physics_bear) and hyper_pd_bear_ok and not atr_extreme and idm_bear_ok

var int last_quick_bull_bar = na
var int last_quick_bear_bar = na
bool quick_bull = quick_base_bull and (na(last_quick_bull_bar) or bar_index - last_quick_bull_bar > cooldown_bars)
bool quick_bear = quick_base_bear and (na(last_quick_bear_bar) or bar_index - last_quick_bear_bar > cooldown_bars)
if quick_bull
    last_quick_bull_bar := bar_index
if quick_bear
    last_quick_bear_bar := bar_index

bool strong_bull = strong_base_bull and setup_id_bull != consumed_setup_bull
bool strong_bear = strong_base_bear and setup_id_bear != consumed_setup_bear
bool hyper_bull = hyper_base_bull and setup_id_bull != consumed_hyper_bull
bool hyper_bear = hyper_base_bear and setup_id_bear != consumed_hyper_bear

bool htf_long_blocked = (block_long_when_htf_bear and use_htf_trend and htf_trend_down) or hard_block_long
bool htf_short_blocked = (block_short_when_htf_bull and use_htf_trend and htf_trend_up) or hard_block_short
bool score_long_blocked = use_score_bias_filter and bear_score > bull_score + score_bias_gap
bool score_short_blocked = use_score_bias_filter and bull_score > bear_score + score_bias_gap
bool market_signal_blocked = not confirmed_market_ok

bool long_signal_allowed = not market_signal_blocked and not htf_long_blocked and not score_long_blocked
bool short_signal_allowed = not market_signal_blocked and not htf_short_blocked and not score_short_blocked

bool quick_bull_blocked = enable_quick and quick_bull and not long_signal_allowed
bool quick_bear_blocked = enable_quick and quick_bear and not short_signal_allowed
bool strong_bull_blocked = enable_strong and strong_bull and not long_signal_allowed
bool strong_bear_blocked = enable_strong and strong_bear and not short_signal_allowed
bool hyper_bull_blocked = enable_hyper and hyper_bull and not long_signal_allowed
bool hyper_bear_blocked = enable_hyper and hyper_bear and not short_signal_allowed

bool hyper_bull_final = enable_hyper and hyper_bull and long_signal_allowed
bool hyper_bear_final = enable_hyper and hyper_bear and short_signal_allowed
bool strong_bull_final = enable_strong and strong_bull and long_signal_allowed and not hyper_bull_final
bool strong_bear_final = enable_strong and strong_bear and short_signal_allowed and not hyper_bear_final
bool quick_bull_final = enable_quick and quick_bull and long_signal_allowed and not strong_bull_final and not hyper_bull_final
bool quick_bear_final = enable_quick and quick_bear and short_signal_allowed and not strong_bear_final and not hyper_bear_final

if strong_bull_final
    consumed_setup_bull := setup_id_bull
if strong_bear_final
    consumed_setup_bear := setup_id_bear
if hyper_bull_final
    consumed_hyper_bull := setup_id_bull
    consumed_setup_bull := setup_id_bull
if hyper_bear_final
    consumed_hyper_bear := setup_id_bear
    consumed_setup_bear := setup_id_bear

// =============================================================================
// L15: REALTIME EARLY SIGNALS
// =============================================================================
float realtime_prev_high = ta.highest(high[1], realtime_lookback)
float realtime_prev_low = ta.lowest(low[1], realtime_lookback)
bool realtime_break_up = use_realtime_signals and not na(realtime_prev_high) and high > realtime_prev_high
bool realtime_break_down = use_realtime_signals and not na(realtime_prev_low) and low < realtime_prev_low
bool realtime_reclaim_up = realtime_break_down and close > realtime_prev_low
bool realtime_reclaim_down = realtime_break_up and close < realtime_prev_high

bool realtime_rejection_bull_ok = not realtime_require_rejection or bull_rejection >= sweep_reject_min or close > open
bool realtime_rejection_bear_ok = not realtime_require_rejection or bear_rejection >= sweep_reject_min or close < open
bool realtime_htf_bull_ok = not realtime_require_htf or htf_trend_up
bool realtime_htf_bear_ok = not realtime_require_htf or htf_trend_down
bool realtime_momentum_bull_ok = not realtime_require_momentum or momentum_bull or physics_bull or fast_momentum_up
bool realtime_momentum_bear_ok = not realtime_require_momentum or momentum_bear or physics_bear or fast_momentum_down
bool realtime_bar_ok = show_realtime_on_history or not confirmed

var int rt_bull_commit = 0
var int rt_bear_commit = 0
if realtime_reclaim_up or realtime_break_up
    rt_bull_commit += 1
else
    rt_bull_commit := 0
if realtime_reclaim_down or realtime_break_down
    rt_bear_commit += 1
else
    rt_bear_commit := 0

bool realtime_bull_locked = rt_bull_commit >= 1
bool realtime_bear_locked = rt_bear_commit >= 1

int realtime_bull_score = 0
int realtime_bear_score = 0
realtime_bull_score += realtime_reclaim_up ? 25 : realtime_break_up ? 12 : 0
realtime_bull_score += realtime_momentum_bull_ok ? 20 : 0
realtime_bull_score += realtime_htf_bull_ok ? 15 : 0
realtime_bull_score += in_discount or in_bull_ote ? 12 : 0
realtime_bull_score += bull_rr_ok ? 10 : 0
realtime_bull_score += vol_ok ? 8 : 0
realtime_bull_score += physics_bull ? 10 : 0
realtime_bull_score := math.min(realtime_bull_score, 100)

realtime_bear_score += realtime_reclaim_down ? 25 : realtime_break_down ? 12 : 0
realtime_bear_score += realtime_momentum_bear_ok ? 20 : 0
realtime_bear_score += realtime_htf_bear_ok ? 15 : 0
realtime_bear_score += in_premium or in_bear_ote ? 12 : 0
realtime_bear_score += bear_rr_ok ? 10 : 0
realtime_bear_score += vol_ok ? 8 : 0
realtime_bear_score += physics_bear ? 10 : 0
realtime_bear_score := math.min(realtime_bear_score, 100)

bool realtime_liquidity_bull = (realtime_mode == "كسر سيولة" or realtime_mode == "كلاهما") and (realtime_break_up or realtime_reclaim_up)
bool realtime_liquidity_bear = (realtime_mode == "كسر سيولة" or realtime_mode == "كلاهما") and (realtime_break_down or realtime_reclaim_down)
bool realtime_direct_bull = (realtime_mode == "درجة مباشرة" or realtime_mode == "كلاهما") and bull_score >= profile_realtime_score_min and bull_score > bear_score + score_bias_gap
bool realtime_direct_bear = (realtime_mode == "درجة مباشرة" or realtime_mode == "كلاهما") and bear_score >= profile_realtime_score_min and bear_score > bull_score + score_bias_gap

bool realtime_bull_final = use_realtime_signals and show_realtime_signals and realtime_bar_ok and long_signal_allowed and (realtime_liquidity_bull or realtime_direct_bull) and realtime_rejection_bull_ok and realtime_momentum_bull_ok and realtime_htf_bull_ok and realtime_bull_locked
bool realtime_bear_final = use_realtime_signals and show_realtime_signals and realtime_bar_ok and short_signal_allowed and (realtime_liquidity_bear or realtime_direct_bear) and realtime_rejection_bear_ok and realtime_momentum_bear_ok and realtime_htf_bear_ok and realtime_bear_locked

bool watch_bull = show_decision_stage and show_realtime_signals and not realtime_bull_final and not quick_bull_final and not strong_bull_final and not hyper_bull_final and long_signal_allowed and bull_score >= math.max(35, profile_realtime_score_min - 12) and bull_score > bear_score
bool watch_bear = show_decision_stage and show_realtime_signals and not realtime_bear_final and not quick_bear_final and not strong_bear_final and not hyper_bear_final and short_signal_allowed and bear_score >= math.max(35, profile_realtime_score_min - 12) and bear_score > bull_score
// =============================================================================
// L21: SETUP RECORD TYPE
// =============================================================================
type SetupRecord
    int id
    int born_bar
    int entry_bar
    int direction
    int signal_type
    float entry
    float sl
    float tp
    float initial_risk
    float planned_rr
    int score
    int regime
    int session
    int structure_score
    int liquidity_score
    int zone_score
    int location_score
    int momentum_score
    int htf_score
    float mfe_r
    float mae_r
    int best_bar
    int worst_bar
    bool tp_hit
    bool sl_hit
    bool both_hit
    bool resolved
    bool expired
    int resolution
    int exit_bar
    float realized_r
    int duration
    bool sweep
    bool bos
    bool choch
    bool ob
    bool fvg
    bool ote
    bool discount
    bool momentum
    bool physics
    bool htf_flag
    bool poc
    float snapshot_atr
    float snapshot_vol_ratio
    float snapshot_htf_close
    float snapshot_htf_ema
    bool inv_fvg
    float cost_r

var array<SetupRecord> active_trades = array.new<SetupRecord>()
var array<SetupRecord> closed_trades = array.new<SetupRecord>()
var int global_setup_id = 0

// =============================================================================
// V13 STEP 9: RESEARCH EXPORT BRIDGE — SERIALIZATION
// =============================================================================
f_json_float(float x) =>
    string out = na(x) ? "null" : str.tostring(x, "#.######")
    out

f_json_price(float x) =>
    string out = na(x) ? "null" : str.tostring(x, format.mintick)
    out

f_json_int(int x) =>
    string out = na(x) ? "null" : str.tostring(x)
    out

f_bool_int(bool x) =>
    x ? 1 : 0

f_export_message(SetupRecord rec) =>
    string direction = rec.direction == 1 ? "LONG" : "SHORT"
    string sig_type = rec.signal_type == 2 ? "فائق" : rec.signal_type == 1 ? "قوي" : "سريع"
    string res_text = rec.resolution == 1 ? "TP" : rec.resolution == 2 ? "SL" : rec.resolution == 3 ? "AMBIGUOUS" : rec.resolution == 4 ? "TIME" : "UNKNOWN"

    string msg = ""

    if export_format == "CSV"
        msg := str.tostring(time) + ","
        msg := msg + '"' + syminfo.ticker + '",'
        msg := msg + timeframe.period + ","
        msg := msg + str.tostring(rec.id) + ","
        msg := msg + f_json_int(rec.entry_bar) + ","
        msg := msg + f_json_int(rec.exit_bar) + ","
        msg := msg + direction + ","
        msg := msg + sig_type + ","
        msg := msg + str.tostring(rec.score) + ","
        msg := msg + str.tostring(rec.regime) + ","
        msg := msg + str.tostring(rec.session) + ","
        msg := msg + f_json_price(rec.entry) + ","
        msg := msg + f_json_price(rec.sl) + ","
        msg := msg + f_json_price(rec.tp) + ","
        msg := msg + f_json_float(rec.initial_risk) + ","
        msg := msg + f_json_float(rec.planned_rr) + ","
        msg := msg + f_json_float(rec.cost_r) + ","
        msg := msg + res_text + ","
        msg := msg + f_json_float(rec.realized_r) + ","
        msg := msg + f_json_float(rec.mfe_r) + ","
        msg := msg + f_json_float(rec.mae_r) + ","
        msg := msg + str.tostring(rec.duration) + ","
        msg := msg + str.tostring(rec.structure_score) + ","
        msg := msg + str.tostring(rec.liquidity_score) + ","
        msg := msg + str.tostring(rec.zone_score) + ","
        msg := msg + str.tostring(rec.location_score) + ","
        msg := msg + str.tostring(rec.momentum_score) + ","
        msg := msg + str.tostring(rec.htf_score) + ","
        msg := msg + str.tostring(f_bool_int(rec.sweep)) + ","
        msg := msg + str.tostring(f_bool_int(rec.bos)) + ","
        msg := msg + str.tostring(f_bool_int(rec.choch)) + ","
        msg := msg + str.tostring(f_bool_int(rec.ob)) + ","
        msg := msg + str.tostring(f_bool_int(rec.fvg)) + ","
        msg := msg + str.tostring(f_bool_int(rec.ote)) + ","
        msg := msg + str.tostring(f_bool_int(rec.discount)) + ","
        msg := msg + str.tostring(f_bool_int(rec.momentum)) + ","
        msg := msg + str.tostring(f_bool_int(rec.physics)) + ","
        msg := msg + str.tostring(f_bool_int(rec.htf_flag)) + ","
        msg := msg + str.tostring(f_bool_int(rec.poc)) + ","
        msg := msg + str.tostring(f_bool_int(rec.inv_fvg)) + ","
        msg := msg + str.tostring(f_bool_int(rec.both_hit))
        if export_include_snapshots
            msg := msg + "," + f_json_float(rec.snapshot_atr)
            msg := msg + "," + f_json_float(rec.snapshot_vol_ratio)
            msg := msg + "," + f_json_price(rec.snapshot_htf_close)
            msg := msg + "," + f_json_price(rec.snapshot_htf_ema)
    else
        msg := msg + '{"engine":"APEXv13",'
        msg := msg + '"event":"closed_trade",'
        msg := msg + '"time":' + str.tostring(time) + ','
        msg := msg + '"symbol":"' + syminfo.ticker + '",'
        msg := msg + '"tf":"' + timeframe.period + '",'
        msg := msg + '"id":' + str.tostring(rec.id) + ','
        msg := msg + '"entry_bar":' + f_json_int(rec.entry_bar) + ','
        msg := msg + '"exit_bar":' + f_json_int(rec.exit_bar) + ','
        msg := msg + '"dir":"' + direction + '",'
        msg := msg + '"type":"' + sig_type + '",'
        msg := msg + '"score":' + str.tostring(rec.score) + ','
        msg := msg + '"regime":' + str.tostring(rec.regime) + ','
        msg := msg + '"session":' + str.tostring(rec.session) + ','
        msg := msg + '"entry":' + f_json_price(rec.entry) + ','
        msg := msg + '"sl":' + f_json_price(rec.sl) + ','
        msg := msg + '"tp":' + f_json_price(rec.tp) + ','
        msg := msg + '"risk":' + f_json_float(rec.initial_risk) + ','
        msg := msg + '"planned_rr":' + f_json_float(rec.planned_rr) + ','
        msg := msg + '"cost_r":' + f_json_float(rec.cost_r) + ','
        msg := msg + '"resolution":"' + res_text + '",'
        msg := msg + '"realized_r":' + f_json_float(rec.realized_r) + ','
        msg := msg + '"mfe_r":' + f_json_float(rec.mfe_r) + ','
        msg := msg + '"mae_r":' + f_json_float(rec.mae_r) + ','
        msg := msg + '"duration":' + str.tostring(rec.duration) + ','

        msg := msg + '"scores":{'
        msg := msg + '"structure":' + str.tostring(rec.structure_score) + ','
        msg := msg + '"liquidity":' + str.tostring(rec.liquidity_score) + ','
        msg := msg + '"zone":' + str.tostring(rec.zone_score) + ','
        msg := msg + '"location":' + str.tostring(rec.location_score) + ','
        msg := msg + '"momentum":' + str.tostring(rec.momentum_score) + ','
        msg := msg + '"htf":' + str.tostring(rec.htf_score)
        msg := msg + '},'

        msg := msg + '"flags":{'
        msg := msg + '"sweep":' + str.tostring(f_bool_int(rec.sweep)) + ','
        msg := msg + '"bos":' + str.tostring(f_bool_int(rec.bos)) + ','
        msg := msg + '"choch":' + str.tostring(f_bool_int(rec.choch)) + ','
        msg := msg + '"ob":' + str.tostring(f_bool_int(rec.ob)) + ','
        msg := msg + '"fvg":' + str.tostring(f_bool_int(rec.fvg)) + ','
        msg := msg + '"ote":' + str.tostring(f_bool_int(rec.ote)) + ','
        msg := msg + '"discount":' + str.tostring(f_bool_int(rec.discount)) + ','
        msg := msg + '"momentum":' + str.tostring(f_bool_int(rec.momentum)) + ','
        msg := msg + '"physics":' + str.tostring(f_bool_int(rec.physics)) + ','
        msg := msg + '"htf":' + str.tostring(f_bool_int(rec.htf_flag)) + ','
        msg := msg + '"poc":' + str.tostring(f_bool_int(rec.poc)) + ','
        msg := msg + '"inv_fvg":' + str.tostring(f_bool_int(rec.inv_fvg)) + ','
        msg := msg + '"both_hit":' + str.tostring(f_bool_int(rec.both_hit))
        msg := msg + '}'

        if export_include_snapshots
            msg := msg + ',"snapshot":{'
            msg := msg + '"atr":' + f_json_float(rec.snapshot_atr) + ','
            msg := msg + '"vol_ratio":' + f_json_float(rec.snapshot_vol_ratio) + ','
            msg := msg + '"htf_close":' + f_json_price(rec.snapshot_htf_close) + ','
            msg := msg + '"htf_ema":' + f_json_price(rec.snapshot_htf_ema)
            msg := msg + '}'

        msg := msg + '}'

    msg

// =============================================================================
// L22: SIGNAL SNAPSHOT & PENDING ENGINE
// =============================================================================
// NOTE: cost_r is already computed in Batch 2 via Dynamic Execution Cost Model.
// The original static cost_r calculation has been removed to avoid duplication.

var bool has_pending_bull = false
var int pending_bull_id = na
var int pending_bull_signal_type = na
var int pending_bull_score = na
var float pending_bull_sl = na
var float pending_bull_tp = na
var float pending_bull_planned_rr = na
var int pending_bull_regime = na
var int pending_bull_session = na
var int pending_bull_structure_score = na
var int pending_bull_liquidity_score = na
var int pending_bull_zone_score = na
var int pending_bull_location_score = na
var int pending_bull_momentum_score = na
var int pending_bull_htf_score = na
var bool pending_bull_sweep = false
var bool pending_bull_bos = false
var bool pending_bull_choch = false
var bool pending_bull_ob = false
var bool pending_bull_fvg = false
var bool pending_bull_ote = false
var bool pending_bull_discount = false
var bool pending_bull_momentum = false
var bool pending_bull_physics = false
var bool pending_bull_htf_flag = false
var bool pending_bull_poc = false
var int pending_bull_bar = na
var float pending_bull_atr = na
var float pending_bull_vol_ratio = na
var float pending_bull_htf_close = na
var float pending_bull_htf_ema = na
var bool pending_bull_inv_fvg = false

var bool has_pending_bear = false
var int pending_bear_id = na
var int pending_bear_signal_type = na
var int pending_bear_score = na
var float pending_bear_sl = na
var float pending_bear_tp = na
var float pending_bear_planned_rr = na
var int pending_bear_regime = na
var int pending_bear_session = na
var int pending_bear_structure_score = na
var int pending_bear_liquidity_score = na
var int pending_bear_zone_score = na
var int pending_bear_location_score = na
var int pending_bear_momentum_score = na
var int pending_bear_htf_score = na
var bool pending_bear_sweep = false
var bool pending_bear_bos = false
var bool pending_bear_choch = false
var bool pending_bear_ob = false
var bool pending_bear_fvg = false
var bool pending_bear_ote = false
var bool pending_bear_premium = false
var bool pending_bear_momentum = false
var bool pending_bear_physics = false
var bool pending_bear_htf_flag = false
var bool pending_bear_poc = false
var int pending_bear_bar = na
var float pending_bear_atr = na
var float pending_bear_vol_ratio = na
var float pending_bear_htf_close = na
var float pending_bear_htf_ema = na
var bool pending_bear_inv_fvg = false

if research_enabled
    if hyper_bull_final or strong_bull_final or quick_bull_final
        global_setup_id += 1
        has_pending_bull := true
        pending_bull_id := global_setup_id
        pending_bull_signal_type := hyper_bull_final ? 2 : strong_bull_final ? 1 : 0
        pending_bull_score := bull_score
        pending_bull_sl := bull_sl
        pending_bull_tp := bull_tp
        pending_bull_planned_rr := bull_rr
        pending_bull_regime := regime_trend ? 0 : regime_range_ok ? 1 : regime_expansion ? 2 : regime_compression ? 3 : 4
        pending_bull_session := in_session_asia ? 0 : in_session_london ? 1 : in_session_ny ? 2 : 3
        pending_bull_structure_score := struct_bull_score
        pending_bull_liquidity_score := liq_weight_low
        pending_bull_zone_score := zone_bull_score
        pending_bull_location_score := loc_bull_score
        pending_bull_momentum_score := mom_bull_score
        pending_bull_htf_score := htf_bull_score
        pending_bull_sweep := _sweep_ok_bull
        pending_bull_bos := _bos_ok_bull
        pending_bull_choch := _choch_ok_bull
        pending_bull_ob := _ob_ok_bull
        pending_bull_fvg := _fvg_ok_bull
        pending_bull_ote := in_bull_ote
        pending_bull_discount := in_discount
        pending_bull_momentum := momentum_bull
        pending_bull_physics := physics_bull
        pending_bull_htf_flag := htf_power_bull or htf_trend_up
        pending_bull_poc := near_poc
        pending_bull_bar := bar_index
        pending_bull_atr := safe_atr
        pending_bull_vol_ratio := vol_ratio
        pending_bull_htf_close := htf_close
        pending_bull_htf_ema := htf_ema
        pending_bull_inv_fvg := array.size(inv_bull_tops) > 0 ? array.get(inv_bull_valids, 0) : false

    if hyper_bear_final or strong_bear_final or quick_bear_final
        global_setup_id += 1
        has_pending_bear := true
        pending_bear_id := global_setup_id
        pending_bear_signal_type := hyper_bear_final ? 2 : strong_bear_final ? 1 : 0
        pending_bear_score := bear_score
        pending_bear_sl := bear_sl
        pending_bear_tp := bear_tp
        pending_bear_planned_rr := bear_rr
        pending_bear_regime := regime_trend ? 0 : regime_range_ok ? 1 : regime_expansion ? 2 : regime_compression ? 3 : 4
        pending_bear_session := in_session_asia ? 0 : in_session_london ? 1 : in_session_ny ? 2 : 3
        pending_bear_structure_score := struct_bear_score
        pending_bear_liquidity_score := liq_weight_high
        pending_bear_zone_score := zone_bear_score
        pending_bear_location_score := loc_bear_score
        pending_bear_momentum_score := mom_bear_score
        pending_bear_htf_score := htf_bear_score
        pending_bear_sweep := _sweep_ok_bear
        pending_bear_bos := _bos_ok_bear
        pending_bear_choch := _choch_ok_bear
        pending_bear_ob := _ob_ok_bear
        pending_bear_fvg := _fvg_ok_bear
        pending_bear_ote := in_bear_ote
        pending_bear_premium := in_premium
        pending_bear_momentum := momentum_bear
        pending_bear_physics := physics_bear
        pending_bear_htf_flag := htf_power_bear or htf_trend_down
        pending_bear_poc := near_poc
        pending_bear_bar := bar_index
        pending_bear_atr := safe_atr
        pending_bear_vol_ratio := vol_ratio
        pending_bear_htf_close := htf_close
        pending_bear_htf_ema := htf_ema
        pending_bear_inv_fvg := array.size(inv_bear_tops) > 0 ? array.get(inv_bear_valids, 0) : false

var int rejected_trades = 0

// =============================================================================
// L23: ENTRY ENGINE
// =============================================================================
if research_enabled
    if has_pending_bull and bar_index > pending_bull_bar
        float entry_price = entry_mode == "Next Open" ? open : close[1]
        float risk = math.abs(entry_price - pending_bull_sl)
        bool risk_ok = not na(risk) and risk > syminfo.mintick
        if risk_ok
            SetupRecord rec = SetupRecord.new(
                 pending_bull_id, pending_bull_bar, bar_index, 1, pending_bull_signal_type,
                 entry_price, pending_bull_sl, pending_bull_tp, risk, pending_bull_planned_rr,
                 pending_bull_score, pending_bull_regime, pending_bull_session,
                 pending_bull_structure_score, pending_bull_liquidity_score,
                 pending_bull_zone_score, pending_bull_location_score,
                 pending_bull_momentum_score, pending_bull_htf_score,
                 0.0, 0.0, bar_index, bar_index,
                 false, false, false, false, false, 0, na, na, 0,
                 pending_bull_sweep, pending_bull_bos, pending_bull_choch,
                 pending_bull_ob, pending_bull_fvg, pending_bull_ote,
                 pending_bull_discount, pending_bull_momentum, pending_bull_physics,
                 pending_bull_htf_flag, pending_bull_poc,
                 pending_bull_atr, pending_bull_vol_ratio,
                 pending_bull_htf_close, pending_bull_htf_ema,
                 pending_bull_inv_fvg, cost_r)
            if array.size(active_trades) < research_max_active
                array.push(active_trades, rec)
            else
                rejected_trades += 1
        else
            rejected_trades += 1
        has_pending_bull := false

    if has_pending_bear and bar_index > pending_bear_bar
        float entry_price = entry_mode == "Next Open" ? open : close[1]
        float risk = math.abs(entry_price - pending_bear_sl)
        bool risk_ok = not na(risk) and risk > syminfo.mintick
        if risk_ok
            SetupRecord rec = SetupRecord.new(
                 pending_bear_id, pending_bear_bar, bar_index, -1, pending_bear_signal_type,
                 entry_price, pending_bear_sl, pending_bear_tp, risk, pending_bear_planned_rr,
                 pending_bear_score, pending_bear_regime, pending_bear_session,
                 pending_bear_structure_score, pending_bear_liquidity_score,
                 pending_bear_zone_score, pending_bear_location_score,
                 pending_bear_momentum_score, pending_bear_htf_score,
                 0.0, 0.0, bar_index, bar_index,
                 false, false, false, false, false, 0, na, na, 0,
                 pending_bear_sweep, pending_bear_bos, pending_bear_choch,
                 pending_bear_ob, pending_bear_fvg, pending_bear_ote,
                 pending_bear_premium, pending_bear_momentum, pending_bear_physics,
                 pending_bear_htf_flag, pending_bear_poc,
                 pending_bear_atr, pending_bear_vol_ratio,
                 pending_bear_htf_close, pending_bear_htf_ema,
                 pending_bear_inv_fvg, cost_r)
            if array.size(active_trades) < research_max_active
                array.push(active_trades, rec)
            else
                rejected_trades += 1
        else
            rejected_trades += 1
        has_pending_bear := false

// =============================================================================
// V13 STEP 5: INTRABAR EXECUTION ENGINE
// =============================================================================
bool ib_tf_ok = timeframe.in_seconds(intrabar_tf) < timeframe.in_seconds(timeframe.period)
bool intrabar_enabled = use_intrabar_engine and research_enabled and ib_tf_ok and barstate.isconfirmed

var array<float> ib_highs = array.new<float>()
var array<float> ib_lows = array.new<float>()

if intrabar_enabled
    [ib_high_arr, ib_low_arr] = request.security_lower_tf(syminfo.tickerid, intrabar_tf, [high, low])
    ib_highs := ib_high_arr
    ib_lows := ib_low_arr
else
    array.clear(ib_highs)
    array.clear(ib_lows)

// =============================================================================
// L24-L26: ACTIVE OUTCOME ENGINE & RESOLUTION (with V13 Intrabar)
// =============================================================================
if research_enabled
    int active_size = array.size(active_trades)
    if active_size > 0
        for i = 0 to active_size - 1
            SetupRecord rec = array.get(active_trades, i)
            if not rec.resolved and not rec.expired
                float favorable = na
                float adverse = na
                if rec.direction == 1
                    favorable := (high - rec.entry) / rec.initial_risk
                    adverse := (rec.entry - low) / rec.initial_risk
                else
                    favorable := (rec.entry - low) / rec.initial_risk
                    adverse := (high - rec.entry) / rec.initial_risk

                float new_mfe = math.max(rec.mfe_r, nz(favorable, 0))
                float new_mae = math.max(rec.mae_r, nz(adverse, 0))
                int new_best_bar = new_mfe > rec.mfe_r ? bar_index : rec.best_bar
                int new_worst_bar = new_mae > rec.mae_r ? bar_index : rec.worst_bar

                bool hit_tp = false
                bool hit_sl = false
                if rec.direction == 1
                    hit_tp := not na(rec.tp) and high >= rec.tp
                    hit_sl := not na(rec.sl) and low <= rec.sl
                else
                    hit_tp := not na(rec.tp) and low <= rec.tp
                    hit_sl := not na(rec.sl) and high >= rec.sl

                bool new_tp_hit = rec.tp_hit or hit_tp
                bool new_sl_hit = rec.sl_hit or hit_sl
                bool new_both_hit = new_tp_hit and new_sl_hit and not rec.resolved

                int new_resolution = rec.resolution
                bool new_resolved = rec.resolved
                int new_exit_bar = rec.exit_bar
                float new_realized_r = rec.realized_r
                bool new_expired = rec.expired

                // V13 STEP 5: INTRABAR RESOLUTION
                int ib_first_hit = 0
                if new_both_hit and intrabar_enabled and array.size(ib_highs) > 0 and array.size(ib_lows) > 0
                    int ib_size = math.min(array.size(ib_highs), array.size(ib_lows))
                    if ib_size > 0
                        for k = 0 to ib_size - 1
                            float ib_h = array.get(ib_highs, k)
                            float ib_l = array.get(ib_lows, k)
                            bool ib_tp = false
                            bool ib_sl = false
                            if rec.direction == 1
                                ib_tp := not na(rec.tp) and ib_h >= rec.tp
                                ib_sl := not na(rec.sl) and ib_l <= rec.sl
                            else
                                ib_tp := not na(rec.tp) and ib_l <= rec.tp
                                ib_sl := not na(rec.sl) and ib_h >= rec.sl
                            if ib_tp and ib_sl
                                ib_first_hit := 3
                                break
                            else if ib_tp
                                ib_first_hit := 1
                                break
                            else if ib_sl
                                ib_first_hit := 2
                                break

                if not rec.resolved
                    if new_both_hit
                        if ib_first_hit == 1
                            new_resolution := 1
                            float ib_tp_raw_r = rec.direction == 1 ? (rec.tp - rec.entry) / rec.initial_risk : (rec.entry - rec.tp) / rec.initial_risk
                            new_realized_r := ib_tp_raw_r - rec.cost_r
                            new_resolved := true
                            new_exit_bar := bar_index
                        else if ib_first_hit == 2
                            new_resolution := 2
                            new_realized_r := -1.0 - rec.cost_r
                            new_resolved := true
                            new_exit_bar := bar_index
                        else if ambiguous_policy == "Conservative"
                            new_resolution := 2
                            new_realized_r := -1.0 - rec.cost_r
                            new_resolved := true
                            new_exit_bar := bar_index
                        else if ambiguous_policy == "Optimistic"
                            new_resolution := 1
                            float ib_opt_raw_r = rec.direction == 1 ? (rec.tp - rec.entry) / rec.initial_risk : (rec.entry - rec.tp) / rec.initial_risk
                            new_realized_r := ib_opt_raw_r - rec.cost_r
                            new_resolved := true
                            new_exit_bar := bar_index
                        else
                            new_resolution := 3
                            new_realized_r := 0.0 - rec.cost_r
                            new_resolved := true
                            new_exit_bar := bar_index
                    else if new_tp_hit
                        new_resolution := 1
                        float tp_raw_r = rec.direction == 1 ? (rec.tp - rec.entry) / rec.initial_risk : (rec.entry - rec.tp) / rec.initial_risk
                        new_realized_r := tp_raw_r - rec.cost_r
                        new_resolved := true
                        new_exit_bar := bar_index
                    else if new_sl_hit
                        new_resolution := 2
                        float sl_raw_r = rec.direction == 1 ? (rec.sl - rec.entry) / rec.initial_risk : (rec.entry - rec.sl) / rec.initial_risk
                        new_realized_r := sl_raw_r - rec.cost_r
                        new_resolved := true
                        new_exit_bar := bar_index
                    else if bar_index - rec.entry_bar >= max_trade_bars
                        new_expired := true
                        new_resolved := true
                        new_exit_bar := bar_index
                        float exit_price = close
                        float expire_raw_r = rec.direction == 1 ? (exit_price - rec.entry) / rec.initial_risk : (rec.entry - exit_price) / rec.initial_risk
                        new_realized_r := expire_raw_r - rec.cost_r
                        new_resolution := 4

                int new_duration = new_resolved and not na(new_exit_bar) ? new_exit_bar - rec.entry_bar : bar_index - rec.entry_bar

                SetupRecord updated = SetupRecord.new(
                     rec.id, rec.born_bar, rec.entry_bar, rec.direction, rec.signal_type,
                     rec.entry, rec.sl, rec.tp, rec.initial_risk, rec.planned_rr,
                     rec.score, rec.regime, rec.session,
                     rec.structure_score, rec.liquidity_score,
                     rec.zone_score, rec.location_score,
                     rec.momentum_score, rec.htf_score,
                     new_mfe, new_mae, new_best_bar, new_worst_bar,
                     new_tp_hit, new_sl_hit, new_both_hit,
                     new_resolved, new_expired, new_resolution,
                     new_exit_bar, new_realized_r, new_duration,
                     rec.sweep, rec.bos, rec.choch, rec.ob, rec.fvg,
                     rec.ote, rec.discount, rec.momentum, rec.physics,
                     rec.htf_flag, rec.poc,
                     rec.snapshot_atr, rec.snapshot_vol_ratio,
                     rec.snapshot_htf_close, rec.snapshot_htf_ema,
                     rec.inv_fvg, rec.cost_r)
                array.set(active_trades, i, updated)

// =============================================================================
// V13 STEP 3: HISTORICAL PROBABILITY CALIBRATOR — COUNTERS
// =============================================================================
int calib_min_trades = 10
float calib_alpha = 10.0
float calib_beta = 10.0

var int[] calib_trades = array.new_int(6, 0)
var int[] calib_wins = array.new_int(6, 0)
var float[] calib_sum_r = array.new_float(6, 0.0)
var float[] calib_gp = array.new_float(6, 0.0)
var float[] calib_gl = array.new_float(6, 0.0)

f_calib_bin(int s) =>
    int b = 0
    if s >= 93
        b := 5
    else if s >= 85
        b := 4
    else if s >= 75
        b := 3
    else if s >= 65
        b := 2
    else if s >= 55
        b := 1
    else
        b := 0
    b

f_calib_prob(int bin) =>
    int n = array.get(calib_trades, bin)
    int w = array.get(calib_wins, bin)
    float prob = n >= calib_min_trades ? (w + calib_alpha) / (n + calib_alpha + calib_beta) * 100.0 : na
    prob

f_fmt(float x, string fmt) =>
    string out = na(x) ? "-" : str.tostring(x, fmt)
    out

// =============================================================================
// V13 STEP 7: SIGNAL QUALITY MATRIX — COUNTERS
// =============================================================================
int sqm_bins = 6
int sqm_types = 3
int sqm_regimes = 5
int sqm_size = sqm_bins * sqm_types

float sqm_alpha = 10.0
float sqm_beta = 10.0

var int[] sqm_trades = array.new_int(sqm_size, 0)
var int[] sqm_wins = array.new_int(sqm_size, 0)
var float[] sqm_sum_r = array.new_float(sqm_size, 0.0)
var float[] sqm_gp = array.new_float(sqm_size, 0.0)
var float[] sqm_gl = array.new_float(sqm_size, 0.0)

var int[] sqm_reg_trades = array.new_int(sqm_regimes, 0)
var int[] sqm_reg_wins = array.new_int(sqm_regimes, 0)
var float[] sqm_reg_sum_r = array.new_float(sqm_regimes, 0.0)

f_sqm_bin(int s) =>
    int b = 0
    if s >= 93
        b := 5
    else if s >= 85
        b := 4
    else if s >= 75
        b := 3
    else if s >= 65
        b := 2
    else if s >= 55
        b := 1
    else
        b := 0
    b

f_sqm_fmt(float x, string fmt) =>
    string out = na(x) ? "-" : str.tostring(x, fmt)
    out

f_sqm_cell(int idx) =>
    int n = array.get(sqm_trades, idx)
    float exp = n > 0 ? array.get(sqm_sum_r, idx) / n : na
    string txt = n == 0 ? "-" : str.tostring(n) + " | " + (na(exp) ? "-" : str.tostring(exp, "#.02") + "R")
    color col = n == 0 ? #8B98A5 : exp > 0 ? #26A69A : exp < 0 ? #EF5350 : #F2C94C
    [txt, col]

// =============================================================================
// L27: CLOSED TRADE STORAGE & STATISTICS ENGINE
// (with V13 Calibration + SQM + Export updates)
// =============================================================================
var int total_trades = 0
var int wins = 0
var int losses = 0
var float gross_profit_r = 0.0
var float gross_loss_r = 0.0
var float sum_r = 0.0
var float sum_mfe = 0.0
var float sum_mae = 0.0
var float equity_r = 0.0
var float peak_equity = 0.0
var float max_dd = 0.0
var int quick_trades = 0
var int quick_wins = 0
var float quick_sum_r = 0.0
var int strong_trades = 0
var int strong_wins = 0
var float strong_sum_r = 0.0
var int hyper_trades = 0
var int hyper_wins = 0
var float hyper_sum_r = 0.0
var int long_trades = 0
var int long_wins = 0
var float long_sum_r = 0.0
var int short_trades = 0
var int short_wins = 0
var float short_sum_r = 0.0

if research_enabled
    int j = 0
    while j < array.size(active_trades)
        SetupRecord rec = array.get(active_trades, j)
        if rec.resolved
            if array.size(closed_trades) >= research_max_closed
                array.shift(closed_trades)
            array.push(closed_trades, rec)
            array.remove(active_trades, j)

            // V13 STEP 3: CALIBRATION UPDATE
            int calib_bin = f_calib_bin(rec.score)
            array.set(calib_trades, calib_bin, array.get(calib_trades, calib_bin) + 1)
            array.set(calib_sum_r, calib_bin, array.get(calib_sum_r, calib_bin) + rec.realized_r)
            if rec.realized_r > 0
                array.set(calib_wins, calib_bin, array.get(calib_wins, calib_bin) + 1)
                array.set(calib_gp, calib_bin, array.get(calib_gp, calib_bin) + rec.realized_r)
            else if rec.realized_r < 0
                array.set(calib_gl, calib_bin, array.get(calib_gl, calib_bin) + math.abs(rec.realized_r))

            // V13 STEP 7: SQM UPDATE
            int sqm_bin = f_sqm_bin(rec.score)
            int sqm_type = rec.signal_type >= 0 and rec.signal_type <= 2 ? rec.signal_type : 0
            int sqm_idx = sqm_bin * sqm_types + sqm_type
            array.set(sqm_trades, sqm_idx, array.get(sqm_trades, sqm_idx) + 1)
            array.set(sqm_sum_r, sqm_idx, array.get(sqm_sum_r, sqm_idx) + rec.realized_r)
            if rec.realized_r > 0
                array.set(sqm_wins, sqm_idx, array.get(sqm_wins, sqm_idx) + 1)
                array.set(sqm_gp, sqm_idx, array.get(sqm_gp, sqm_idx) + rec.realized_r)
            else if rec.realized_r < 0
                array.set(sqm_gl, sqm_idx, array.get(sqm_gl, sqm_idx) + math.abs(rec.realized_r))

            int sqm_reg_idx = rec.regime >= 0 and rec.regime < sqm_regimes ? rec.regime : sqm_regimes - 1
            array.set(sqm_reg_trades, sqm_reg_idx, array.get(sqm_reg_trades, sqm_reg_idx) + 1)
            array.set(sqm_reg_sum_r, sqm_reg_idx, array.get(sqm_reg_sum_r, sqm_reg_idx) + rec.realized_r)
            if rec.realized_r > 0
                array.set(sqm_reg_wins, sqm_reg_idx, array.get(sqm_reg_wins, sqm_reg_idx) + 1)

            // V13 STEP 9: EXPORT
            if use_export_bridge and export_closed_trades and (not export_on_confirmed_only or barstate.isconfirmed)
                alert(f_export_message(rec), alert.freq_all)

            // ORIGINAL STATISTICS
            total_trades += 1
            sum_r += rec.realized_r
            sum_mfe += rec.mfe_r
            sum_mae += rec.mae_r

            if rec.realized_r > 0
                wins += 1
                gross_profit_r += rec.realized_r
            else if rec.realized_r < 0
                losses += 1
                gross_loss_r += math.abs(rec.realized_r)

            equity_r += rec.realized_r
            peak_equity := math.max(peak_equity, equity_r)
            float current_dd = peak_equity - equity_r
            max_dd := math.max(max_dd, current_dd)

            if rec.signal_type == 0
                quick_trades += 1
                quick_sum_r += rec.realized_r
                if rec.realized_r > 0
                    quick_wins += 1
            else if rec.signal_type == 1
                strong_trades += 1
                strong_sum_r += rec.realized_r
                if rec.realized_r > 0
                    strong_wins += 1
            else if rec.signal_type == 2
                hyper_trades += 1
                hyper_sum_r += rec.realized_r
                if rec.realized_r > 0
                    hyper_wins += 1

            if rec.direction == 1
                long_trades += 1
                long_sum_r += rec.realized_r
                if rec.realized_r > 0
                    long_wins += 1
            else
                short_trades += 1
                short_sum_r += rec.realized_r
                if rec.realized_r > 0
                    short_wins += 1
        else
            j += 1

// =============================================================================
// L28: STATISTICS ENGINE
// =============================================================================
float expectancy = total_trades > 0 ? sum_r / total_trades : na
float win_rate = total_trades > 0 ? wins / total_trades * 100.0 : na
float profit_factor = gross_loss_r > 0 ? gross_profit_r / gross_loss_r : na
float avg_win = wins > 0 ? gross_profit_r / wins : na
float avg_loss = losses > 0 ? gross_loss_r / losses : na
float payoff_ratio = avg_loss > 0 ? avg_win / avg_loss : na
float avg_mfe = total_trades > 0 ? sum_mfe / total_trades : na
float avg_mae = total_trades > 0 ? sum_mae / total_trades : na

// =============================================================================
// V13 STEP 3: CURRENT CALIBRATED PROBABILITY
// =============================================================================
int bull_calib_bin = f_calib_bin(bull_score)
int bear_calib_bin = f_calib_bin(bear_score)

float bull_calibrated_prob = f_calib_prob(bull_calib_bin)
float bear_calibrated_prob = f_calib_prob(bear_calib_bin)

// =============================================================================
// L29: GROUP ANALYTICS
// =============================================================================
float quick_win_rate = quick_trades > 0 ? quick_wins / quick_trades * 100.0 : na
float quick_expectancy = quick_trades > 0 ? quick_sum_r / quick_trades : na
float strong_win_rate = strong_trades > 0 ? strong_wins / strong_trades * 100.0 : na
float strong_expectancy = strong_trades > 0 ? strong_sum_r / strong_trades : na
float hyper_win_rate = hyper_trades > 0 ? hyper_wins / hyper_trades * 100.0 : na
float hyper_expectancy = hyper_trades > 0 ? hyper_sum_r / hyper_trades : na
float long_win_rate = long_trades > 0 ? long_wins / long_trades * 100.0 : na
float long_expectancy = long_trades > 0 ? long_sum_r / long_trades : na
float short_win_rate = short_trades > 0 ? short_wins / short_trades * 100.0 : na
float short_expectancy = short_trades > 0 ? short_sum_r / short_trades : na

// =============================================================================
// V13 STEP 8: ANTI-OVERFITTING & STABILITY ENGINE
// =============================================================================
f_stab_bin(int s) =>
    int b = 0
    if s >= 93
        b := 5
    else if s >= 85
        b := 4
    else if s >= 75
        b := 3
    else if s >= 65
        b := 2
    else if s >= 55
        b := 1
    else
        b := 0
    b

if use_stability_engine and research_enabled and barstate.islast
    int n_closed = array.size(closed_trades)

    float recent_exp = na
    float older_exp = na
    int positive_regimes = 0
    int positive_types = 0
    float high_score_exp = na
    float low_score_exp = na
    bool regime_dependent = false
    string stability_warning = "OK"

    float sample_score = math.min(total_trades / float(stability_min_trades), 1.0) * 20.0
    float consistency_score = 0.0
    float regime_score = 0.0
    float type_score = 0.0
    float monotonic_score = 10.0
    float stability_score_local = 0.0

    if n_closed > 0
        int recent_start = math.max(0, n_closed - stability_recent_len)

        float recent_sum = 0.0
        int recent_n = 0
        for i = recent_start to n_closed - 1
            SetupRecord rec = array.get(closed_trades, i)
            recent_n += 1
            recent_sum += rec.realized_r
        recent_exp := recent_n > 0 ? recent_sum / recent_n : na

        float older_sum = 0.0
        int older_n = 0
        if recent_start > 0
            for i = 0 to recent_start - 1
                SetupRecord rec = array.get(closed_trades, i)
                older_n += 1
                older_sum += rec.realized_r
        older_exp := older_n > 0 ? older_sum / older_n : na

        array<int> reg_n = array.new_int(5, 0)
        array<float> reg_sum = array.new_float(5, 0.0)
        array<float> reg_pos = array.new_float(5, 0.0)

        array<int> type_n = array.new_int(3, 0)
        array<float> type_sum = array.new_float(3, 0.0)

        array<int> bin_n = array.new_int(6, 0)
        array<float> bin_sum = array.new_float(6, 0.0)

        float total_positive_r = 0.0

        for i = 0 to n_closed - 1
            SetupRecord rec = array.get(closed_trades, i)

            int reg = rec.regime >= 0 and rec.regime < 5 ? rec.regime : 4
            int typ = rec.signal_type >= 0 and rec.signal_type < 3 ? rec.signal_type : 0
            int bin = f_stab_bin(rec.score)

            array.set(reg_n, reg, array.get(reg_n, reg) + 1)
            array.set(reg_sum, reg, array.get(reg_sum, reg) + rec.realized_r)

            if rec.realized_r > 0
                array.set(reg_pos, reg, array.get(reg_pos, reg) + rec.realized_r)
                total_positive_r += rec.realized_r

            array.set(type_n, typ, array.get(type_n, typ) + 1)
            array.set(type_sum, typ, array.get(type_sum, typ) + rec.realized_r)

            array.set(bin_n, bin, array.get(bin_n, bin) + 1)
            array.set(bin_sum, bin, array.get(bin_sum, bin) + rec.realized_r)

        int min_group_trades = math.max(3, int(math.round(stability_min_trades / 10.0)))

        for r = 0 to 4
            if array.get(reg_n, r) >= min_group_trades and array.get(reg_sum, r) > 0
                positive_regimes += 1

        for t = 0 to 2
            if array.get(type_n, t) >= min_group_trades and array.get(type_sum, t) > 0
                positive_types += 1

        float max_reg_pos = 0.0
        for r = 0 to 4
            max_reg_pos := math.max(max_reg_pos, array.get(reg_pos, r))

        regime_dependent := total_positive_r > 0 and max_reg_pos / total_positive_r > 0.70

        float low_sum = 0.0
        int low_n = 0
        float high_sum = 0.0
        int high_n = 0

        for b = 0 to 5
            int bn = array.get(bin_n, b)
            if bn > 0
                float bs = array.get(bin_sum, b)
                if b <= 2
                    low_n += bn
                    low_sum += bs
                else
                    high_n += bn
                    high_sum += bs

        low_score_exp := low_n > 0 ? low_sum / low_n : na
        high_score_exp := high_n > 0 ? high_sum / high_n : na

        if not na(recent_exp) and not na(older_exp) and recent_n > 0 and older_n > 0
            if recent_exp > 0 and older_exp > 0
                float min_exp = math.min(recent_exp, older_exp)
                float max_exp = math.max(recent_exp, older_exp)
                float ratio = max_exp > 0 ? min_exp / max_exp : 0.0
                consistency_score := 10.0 + ratio * 15.0
            else if recent_exp > 0 or older_exp > 0
                consistency_score := 8.0
            else
                consistency_score := 0.0
        else
            consistency_score := total_trades >= stability_min_trades ? 5.0 : 10.0

        regime_score := positive_regimes >= 2 ? 20.0 : positive_regimes == 1 ? 8.0 : 0.0
        type_score := positive_types >= 2 ? 15.0 : positive_types == 1 ? 6.0 : 0.0

        if not na(high_score_exp) and not na(low_score_exp)
            if high_score_exp > low_score_exp and high_score_exp > 0
                monotonic_score := 20.0
            else if high_score_exp > low_score_exp
                monotonic_score := 12.0
            else if high_score_exp > 0
                monotonic_score := 7.0
            else
                monotonic_score := 2.0
        else
            monotonic_score := 10.0

        stability_score_local := math.min(sample_score + consistency_score + regime_score + type_score + monotonic_score, 100.0)

        if total_trades < stability_min_trades
            stability_warning := "SAMPLE TOO SMALL"
        else if not na(recent_exp) and not na(older_exp) and recent_exp < 0 and older_exp > 0
            stability_warning := "RECENT DECAY"
        else if regime_dependent
            stability_warning := "REGIME DEPENDENT"
        else if not na(high_score_exp) and not na(low_score_exp) and high_score_exp <= low_score_exp
            stability_warning := "SCORE NOT MONOTONIC"
        else
            stability_warning := "OK"
    else
        stability_warning := "NO CLOSED TRADES"

    // V13 STEP 8: STABILITY DASHBOARD (inside stability engine block)
    if show_stability_dashboard
        string stability_grade = stability_score_local >= 80 ? "A" : stability_score_local >= 65 ? "B" : stability_score_local >= 50 ? "C" : stability_score_local >= 35 ? "D" : "F"

        color stability_col = stability_score_local >= 80 ? #26A69A : stability_score_local >= 65 ? #4CAF50 : stability_score_local >= 50 ? #F2C94C : stability_score_local >= 35 ? #FF9800 : #EF5350

        var table stab_tbl = table.new(position.middle_right, 2, 9, bgcolor=color.new(#111820, 92), border_width=1, border_color=#202A35)
        table.clear(stab_tbl, 0, 0, 1, 8)

        table.cell(stab_tbl, 0, 0, "🛡 الثبات", text_color=#E6EDF3, bgcolor=#202A35, text_size=size.tiny)
        table.cell(stab_tbl, 1, 0, stability_grade, text_color=stability_col, bgcolor=#202A35, text_size=size.small)

        table.cell(stab_tbl, 0, 1, "العينة", text_color=#8B98A5, text_size=size.tiny)
        table.cell(stab_tbl, 1, 1, str.tostring(total_trades) + " / " + str.tostring(stability_min_trades), text_color=total_trades >= stability_min_trades ? #26A69A : #F2C94C, text_size=size.tiny)

        table.cell(stab_tbl, 0, 2, "حديث | قديم", text_color=#8B98A5, text_size=size.tiny)
        table.cell(stab_tbl, 1, 2, str.tostring(nz(recent_exp, 0.0), "#.02") + "R | " + str.tostring(nz(older_exp, 0.0), "#.02") + "R", text_color=nz(recent_exp, 0.0) > 0 ? #26A69A : #EF5350, text_size=size.tiny)

        table.cell(stab_tbl, 0, 3, "الأنظمة الإيجابية", text_color=#8B98A5, text_size=size.tiny)
        table.cell(stab_tbl, 1, 3, str.tostring(positive_regimes) + "/5" + (regime_dependent ? " | DEP" : ""), text_color=positive_regimes >= 2 ? #26A69A : positive_regimes == 1 ? #F2C94C : #EF5350, text_size=size.tiny)

        table.cell(stab_tbl, 0, 4, "الأنواع الإيجابية", text_color=#8B98A5, text_size=size.tiny)
        table.cell(stab_tbl, 1, 4, str.tostring(positive_types) + "/3", text_color=positive_types >= 2 ? #26A69A : positive_types == 1 ? #F2C94C : #EF5350, text_size=size.tiny)

        table.cell(stab_tbl, 0, 5, "مرتفع مقابل منخفض", text_color=#8B98A5, text_size=size.tiny)
        table.cell(stab_tbl, 1, 5, str.tostring(nz(high_score_exp, 0.0), "#.02") + " vs " + str.tostring(nz(low_score_exp, 0.0), "#.02"), text_color=nz(high_score_exp, 0.0) > nz(low_score_exp, 0.0) ? #26A69A : #F2C94C, text_size=size.tiny)

        table.cell(stab_tbl, 0, 6, "الدرجة", text_color=#8B98A5, text_size=size.tiny)
        table.cell(stab_tbl, 1, 6, str.tostring(stability_score_local, "#.0") + "/100", text_color=stability_col, text_size=size.tiny)

        string stab_bar = ""
        int stab_blocks = int(math.round(stability_score_local / 10.0))
        for sb = 0 to 9
            stab_bar := stab_bar + (sb < stab_blocks ? "█" : "░")
        table.cell(stab_tbl, 0, 7, stab_bar, text_color=stability_col, text_size=size.tiny)
        table.cell(stab_tbl, 1, 7, "", text_color=color.new(color.gray, 100), text_size=size.tiny)

        table.cell(stab_tbl, 0, 8, "تحذير", text_color=#8B98A5, text_size=size.tiny)
        table.cell(stab_tbl, 1, 8, stability_warning, text_color=stability_warning == "OK" ? #26A69A : #F2C94C, text_size=size.tiny)

// =============================================================================
// V13 STEP 10: PRODUCTION READINESS GATE
// =============================================================================
float prod_score = 0.0
string prod_grade = "-"
string prod_decision = "NOT EVALUATED"
color prod_col = #8B98A5
string prod_warning = ""

float prod_sample_score = 0.0
float prod_edge_score = 0.0
float prod_risk_score = 0.0
float prod_quality_score = 0.0
float prod_recency_score = 0.0

if use_production_gate and research_enabled
    prod_sample_score := math.min(total_trades / float(prod_min_trades), 1.0) * 20.0

    if not na(expectancy) and expectancy > 0
        prod_edge_score += 10.0
        if expectancy >= prod_min_expectancy
            prod_edge_score += 5.0

    if not na(profit_factor) and profit_factor > 1.0
        prod_edge_score += 5.0
        if profit_factor >= prod_min_pf
            prod_edge_score += 5.0

    prod_edge_score := math.min(prod_edge_score, 25.0)

    if total_trades > 0 and not na(max_dd) and max_dd > 0
        float recovery_factor = equity_r / max_dd

        if recovery_factor > 1.0
            prod_risk_score += 10.0
        else if recovery_factor > 0.5
            prod_risk_score += 5.0

        if max_dd <= prod_max_dd
            prod_risk_score += 10.0
        else if max_dd <= prod_max_dd * 1.5
            prod_risk_score += 5.0

        if not na(win_rate) and not na(payoff_ratio)
            if win_rate >= 40.0 and payoff_ratio >= 1.0
                prod_risk_score += 5.0

    prod_risk_score := math.min(prod_risk_score, 25.0)

    if total_trades > 0
        int positive_groups = 0

        if quick_trades > 5 and nz(quick_expectancy, 0.0) > 0
            positive_groups += 1
        if strong_trades > 5 and nz(strong_expectancy, 0.0) > 0
            positive_groups += 1
        if hyper_trades > 5 and nz(hyper_expectancy, 0.0) > 0
            positive_groups += 1
        if long_trades > 5 and nz(long_expectancy, 0.0) > 0
            positive_groups += 1
        if short_trades > 5 and nz(short_expectancy, 0.0) > 0
            positive_groups += 1

        prod_quality_score := math.min(positive_groups / 5.0 * 15.0, 15.0)

    if array.size(closed_trades) > 0
        SetupRecord last_rec = array.get(closed_trades, array.size(closed_trades) - 1)
        int bars_since_last = bar_index - nz(last_rec.exit_bar, bar_index)

        if bars_since_last <= 50
            prod_recency_score := 15.0
        else if bars_since_last <= 200
            prod_recency_score := 10.0
        else
            prod_recency_score := 5.0
    else
        prod_recency_score := 0.0

    prod_score := math.min(prod_sample_score + prod_edge_score + prod_risk_score + prod_quality_score + prod_recency_score, 100.0)

    prod_grade := prod_score >= 80 ? "A" : prod_score >= 65 ? "B" : prod_score >= 50 ? "C" : prod_score >= 35 ? "D" : "F"

    prod_col := prod_score >= 80 ? #26A69A : prod_score >= 65 ? #4CAF50 : prod_score >= 50 ? #F2C94C : prod_score >= 35 ? #FF9800 : #EF5350

    bool sample_ok = total_trades >= prod_min_trades
    bool edge_ok = not na(expectancy) and expectancy >= prod_min_expectancy and not na(profit_factor) and profit_factor >= prod_min_pf
    bool risk_ok = not na(max_dd) and max_dd <= prod_max_dd

    if not sample_ok
        prod_decision := "NO-GO: SAMPLE TOO SMALL"
        prod_warning := "Need " + str.tostring(prod_min_trades) + "+ trades"
    else if not edge_ok
        prod_decision := "NO-GO: NO STATISTICAL EDGE"
        prod_warning := "Expectancy / PF below threshold"
    else if not risk_ok
        prod_decision := "NO-GO: RISK TOO HIGH"
        prod_warning := "Max DD exceeds limit"
    else if prod_score >= 65
        prod_decision := "GO: READY FOR FORWARD TEST"
        prod_warning := "Start with reduced size"
    else
        prod_decision := "HOLD: NEEDS IMPROVEMENT"
        prod_warning := "Improve weak components first"
        // =============================================================================
// L32: RESEARCH DASHBOARD
// =============================================================================
if show_research_dashboard and barstate.islast and research_enabled
    var table research_tbl = table.new(position.bottom_left, 4, 12, bgcolor=color.new(#111820, 92), border_width=1, border_color=#202A35)
    float wr_show = nz(win_rate, 0.0)
    float exp_show = nz(expectancy, 0.0)
    float pf_show = nz(profit_factor, 0.0)
    table.clear(research_tbl, 0, 0, 3, 11)
    table.cell(research_tbl, 0, 0, "🔬 البحث والتحليل", text_color=#E6EDF3, bgcolor=#202A35, text_size=size.small)
    table.cell(research_tbl, 0, 1, str.tostring(total_trades), text_color=#E6EDF3, text_size=size.small)
    table.cell(research_tbl, 1, 1, str.tostring(wr_show, "#.0") + "%", text_color=wr_show > 50 ? #26A69A : #EF5350, text_size=size.small)
    table.cell(research_tbl, 2, 1, str.tostring(exp_show, "#.00") + "R", text_color=exp_show > 0 ? #26A69A : #EF5350, text_size=size.small)
    table.cell(research_tbl, 3, 1, str.tostring(pf_show, "#.00"), text_color=pf_show > 1.5 ? #26A69A : pf_show > 1.0 ? #F2C94C : #EF5350, text_size=size.small)
    table.cell(research_tbl, 0, 2, "الصفقات", text_color=#8B98A5, text_size=size.tiny)
    table.cell(research_tbl, 1, 2, "نسبة الفوز", text_color=#8B98A5, text_size=size.tiny)
    table.cell(research_tbl, 2, 2, "التوقع", text_color=#8B98A5, text_size=size.tiny)
    table.cell(research_tbl, 3, 2, "عامل الربح", text_color=#8B98A5, text_size=size.tiny)
    table.cell(research_tbl, 0, 3, "سريع", text_color=#00BCD4, text_size=size.tiny)
    table.cell(research_tbl, 1, 3, str.tostring(quick_trades), text_color=#8B98A5, text_size=size.tiny)
    table.cell(research_tbl, 2, 3, str.tostring(nz(quick_win_rate, 0.0), "#.0") + "%", text_color=#8B98A5, text_size=size.tiny)
    table.cell(research_tbl, 3, 3, str.tostring(nz(quick_expectancy, 0.0), "#.00") + "R", text_color=#8B98A5, text_size=size.tiny)
    table.cell(research_tbl, 0, 4, "قوي", text_color=#F44336, text_size=size.tiny)
    table.cell(research_tbl, 1, 4, str.tostring(strong_trades), text_color=#8B98A5, text_size=size.tiny)
    table.cell(research_tbl, 2, 4, str.tostring(nz(strong_win_rate, 0.0), "#.0") + "%", text_color=#8B98A5, text_size=size.tiny)
    table.cell(research_tbl, 3, 4, str.tostring(nz(strong_expectancy, 0.0), "#.00") + "R", text_color=#8B98A5, text_size=size.tiny)
    table.cell(research_tbl, 0, 5, "فائق", text_color=#9C27B0, text_size=size.tiny)
    table.cell(research_tbl, 1, 5, str.tostring(hyper_trades), text_color=#8B98A5, text_size=size.tiny)
    table.cell(research_tbl, 2, 5, str.tostring(nz(hyper_win_rate, 0.0), "#.0") + "%", text_color=#8B98A5, text_size=size.tiny)
    table.cell(research_tbl, 3, 5, str.tostring(nz(hyper_expectancy, 0.0), "#.00") + "R", text_color=#8B98A5, text_size=size.tiny)
    table.cell(research_tbl, 0, 6, "شراء", text_color=#26A69A, text_size=size.tiny)
    table.cell(research_tbl, 1, 6, str.tostring(long_trades), text_color=#8B98A5, text_size=size.tiny)
    table.cell(research_tbl, 2, 6, str.tostring(nz(long_win_rate, 0.0), "#.0") + "%", text_color=#8B98A5, text_size=size.tiny)
    table.cell(research_tbl, 3, 6, str.tostring(nz(long_expectancy, 0.0), "#.00") + "R", text_color=#8B98A5, text_size=size.tiny)
    table.cell(research_tbl, 0, 7, "بيع", text_color=#EF5350, text_size=size.tiny)
    table.cell(research_tbl, 1, 7, str.tostring(short_trades), text_color=#8B98A5, text_size=size.tiny)
    table.cell(research_tbl, 2, 7, str.tostring(nz(short_win_rate, 0.0), "#.0") + "%", text_color=#8B98A5, text_size=size.tiny)
    table.cell(research_tbl, 3, 7, str.tostring(nz(short_expectancy, 0.0), "#.00") + "R", text_color=#8B98A5, text_size=size.tiny)
    table.cell(research_tbl, 0, 8, "نشطة", text_color=#8B98A5, text_size=size.tiny)
    table.cell(research_tbl, 1, 8, str.tostring(array.size(active_trades)), text_color=#E6EDF3, text_size=size.tiny)
    table.cell(research_tbl, 2, 8, "مرفوضة", text_color=#8B98A5, text_size=size.tiny)
    table.cell(research_tbl, 3, 8, str.tostring(rejected_trades), text_color=#F2C94C, text_size=size.tiny)
    table.cell(research_tbl, 0, 9, "MFE", text_color=#8B98A5, text_size=size.tiny)
    table.cell(research_tbl, 1, 9, str.tostring(nz(avg_mfe, 0.0), "#.00") + "R", text_color=#E6EDF3, text_size=size.tiny)
    table.cell(research_tbl, 2, 9, "MAE", text_color=#8B98A5, text_size=size.tiny)
    table.cell(research_tbl, 3, 9, str.tostring(nz(avg_mae, 0.0), "#.00") + "R", text_color=#E6EDF3, text_size=size.tiny)
    table.cell(research_tbl, 0, 10, "أقصى تراجع", text_color=#8B98A5, text_size=size.tiny)
    table.cell(research_tbl, 1, 10, str.tostring(max_dd, "#.00") + "R", text_color=#EF5350, text_size=size.tiny)
    table.cell(research_tbl, 2, 10, "الرصيد", text_color=#8B98A5, text_size=size.tiny)
    table.cell(research_tbl, 3, 10, str.tostring(equity_r, "#.00") + "R", text_color=equity_r > 0 ? #26A69A : #EF5350, text_size=size.tiny)

// =============================================================================
// V13 STEP 3: CALIBRATION DASHBOARD
// =============================================================================
if show_research_dashboard and barstate.islast and research_enabled
    var table calib_tbl = table.new(position.middle_left, 6, 10, bgcolor=color.new(#111820, 92), border_width=1, border_color=#202A35)
    table.clear(calib_tbl, 0, 0, 5, 9)

    table.cell(calib_tbl, 0, 0, "الفئة", text_color=#E6EDF3, bgcolor=#202A35, text_size=size.tiny)
    table.cell(calib_tbl, 1, 0, "الصفقات", text_color=#E6EDF3, bgcolor=#202A35, text_size=size.tiny)
    table.cell(calib_tbl, 2, 0, "الفوز٪", text_color=#E6EDF3, bgcolor=#202A35, text_size=size.tiny)
    table.cell(calib_tbl, 3, 0, "التوقع", text_color=#E6EDF3, bgcolor=#202A35, text_size=size.tiny)
    table.cell(calib_tbl, 4, 0, "عامل الربح", text_color=#E6EDF3, bgcolor=#202A35, text_size=size.tiny)
    table.cell(calib_tbl, 5, 0, "المعايرة٪", text_color=#E6EDF3, bgcolor=#202A35, text_size=size.tiny)

    for i = 0 to 5
        string bin_label = i == 0 ? "<55" : i == 1 ? "55-64" : i == 2 ? "65-74" : i == 3 ? "75-84" : i == 4 ? "85-92" : "93+"

        int n = array.get(calib_trades, i)
        int w = array.get(calib_wins, i)

        float wr_bin = n > 0 ? w / n * 100.0 : na
        float exp_bin = n > 0 ? array.get(calib_sum_r, i) / n : na

        float gp_bin = array.get(calib_gp, i)
        float gl_bin = array.get(calib_gl, i)
        float pf_bin = gl_bin > 0 ? gp_bin / gl_bin : na

        float prob_bin = f_calib_prob(i)

        int row = i + 1

        color prob_col = na(prob_bin) ? #8B98A5 : prob_bin >= 55.0 ? #26A69A : prob_bin < 45.0 ? #EF5350 : #F2C94C
        color exp_col = na(exp_bin) ? #8B98A5 : exp_bin > 0 ? #26A69A : exp_bin < 0 ? #EF5350 : #F2C94C
        color pf_col = na(pf_bin) ? #8B98A5 : pf_bin > 1.3 ? #26A69A : pf_bin < 1.0 ? #EF5350 : #F2C94C

        table.cell(calib_tbl, 0, row, bin_label, text_color=#E6EDF3, text_size=size.tiny)
        table.cell(calib_tbl, 1, row, str.tostring(n), text_color=#E6EDF3, text_size=size.tiny)
        table.cell(calib_tbl, 2, row, f_fmt(wr_bin, "#.0") + "%", text_color=wr_bin >= 50 ? #26A69A : #EF5350, text_size=size.tiny)
        table.cell(calib_tbl, 3, row, f_fmt(exp_bin, "#.00") + "R", text_color=exp_col, text_size=size.tiny)
        table.cell(calib_tbl, 4, row, f_fmt(pf_bin, "#.00"), text_color=pf_col, text_size=size.tiny)
        table.cell(calib_tbl, 5, row, f_fmt(prob_bin, "#.0") + "%", text_color=prob_col, text_size=size.tiny)

    table.cell(calib_tbl, 0, 7, "صاعد", text_color=#26A69A, bgcolor=color.new(#26A69A, 92), text_size=size.tiny)
    table.cell(calib_tbl, 1, 7, str.tostring(bull_score), text_color=#E6EDF3, text_size=size.tiny)
    table.cell(calib_tbl, 2, 7, "-", text_color=#8B98A5, text_size=size.tiny)
    table.cell(calib_tbl, 3, 7, "-", text_color=#8B98A5, text_size=size.tiny)
    table.cell(calib_tbl, 4, 7, "-", text_color=#8B98A5, text_size=size.tiny)
    table.cell(calib_tbl, 5, 7, f_fmt(bull_calibrated_prob, "#.0") + "%", text_color=na(bull_calibrated_prob) ? #8B98A5 : bull_calibrated_prob >= 55.0 ? #26A69A : bull_calibrated_prob < 45.0 ? #EF5350 : #F2C94C, text_size=size.tiny)

    table.cell(calib_tbl, 0, 8, "هابط", text_color=#EF5350, bgcolor=color.new(#EF5350, 92), text_size=size.tiny)
    table.cell(calib_tbl, 1, 8, str.tostring(bear_score), text_color=#E6EDF3, text_size=size.tiny)
    table.cell(calib_tbl, 2, 8, "-", text_color=#8B98A5, text_size=size.tiny)
    table.cell(calib_tbl, 3, 8, "-", text_color=#8B98A5, text_size=size.tiny)
    table.cell(calib_tbl, 4, 8, "-", text_color=#8B98A5, text_size=size.tiny)
    table.cell(calib_tbl, 5, 8, f_fmt(bear_calibrated_prob, "#.0") + "%", text_color=na(bear_calibrated_prob) ? #8B98A5 : bear_calibrated_prob >= 55.0 ? #26A69A : bear_calibrated_prob < 45.0 ? #EF5350 : #F2C94C, text_size=size.tiny)

// =============================================================================
// V13 STEP 7: SIGNAL QUALITY MATRIX DASHBOARD
// =============================================================================
if show_research_dashboard and barstate.islast and research_enabled
    var table sqm_tbl = table.new(position.top_left, 4, 15, bgcolor=color.new(#111820, 92), border_width=1, border_color=#202A35)
    table.clear(sqm_tbl, 0, 0, 3, 14)

    table.cell(sqm_tbl, 0, 0, "الدرجة", text_color=#E6EDF3, bgcolor=#202A35, text_size=size.tiny)
    table.cell(sqm_tbl, 1, 0, "سريع", text_color=#00BCD4, bgcolor=#202A35, text_size=size.tiny)
    table.cell(sqm_tbl, 2, 0, "قوي", text_color=#F44336, bgcolor=#202A35, text_size=size.tiny)
    table.cell(sqm_tbl, 3, 0, "فائق", text_color=#9C27B0, bgcolor=#202A35, text_size=size.tiny)

    for b = 0 to 5
        string bin_label = b == 0 ? "<55" : b == 1 ? "55-64" : b == 2 ? "65-74" : b == 3 ? "75-84" : b == 4 ? "85-92" : "93+"
        int row = b + 1

        int q_idx = b * sqm_types + 0
        int s_idx = b * sqm_types + 1
        int h_idx = b * sqm_types + 2

        [q_txt, q_col] = f_sqm_cell(q_idx)
        [s_txt, s_col] = f_sqm_cell(s_idx)
        [h_txt, h_col] = f_sqm_cell(h_idx)

        table.cell(sqm_tbl, 0, row, bin_label, text_color=#E6EDF3, text_size=size.tiny)
        table.cell(sqm_tbl, 1, row, q_txt, text_color=q_col, text_size=size.tiny)
        table.cell(sqm_tbl, 2, row, s_txt, text_color=s_col, text_size=size.tiny)
        table.cell(sqm_tbl, 3, row, h_txt, text_color=h_col, text_size=size.tiny)

    int current_score = bull_score >= bear_score ? bull_score : bear_score
    int current_bin = f_sqm_bin(current_score)

    int current_n = 0
    int current_w = 0
    float current_sr = 0.0

    for t = 0 to sqm_types - 1
        int idx = current_bin * sqm_types + t
        current_n += array.get(sqm_trades, idx)
        current_w += array.get(sqm_wins, idx)
        current_sr += array.get(sqm_sum_r, idx)

    float current_exp = current_n > 0 ? current_sr / current_n : na
    float current_prob = current_n >= 10 ? (current_w + sqm_alpha) / (current_n + sqm_alpha + sqm_beta) * 100.0 : na
    color current_exp_col = na(current_exp) ? #8B98A5 : current_exp > 0 ? #26A69A : current_exp < 0 ? #EF5350 : #8B98A5

    string current_bin_label = current_bin == 0 ? "<55" : current_bin == 1 ? "55-64" : current_bin == 2 ? "65-74" : current_bin == 3 ? "75-84" : current_bin == 4 ? "85-92" : "93+"

    table.cell(sqm_tbl, 0, 7, "الحالي", text_color=#E6EDF3, bgcolor=color.new(#202A35, 80), text_size=size.tiny)
    table.cell(sqm_tbl, 1, 7, current_bin_label + " " + (bull_score >= bear_score ? "صاعد" : "هابط"), text_color=bull_score >= bear_score ? #26A69A : #EF5350, text_size=size.tiny)
    table.cell(sqm_tbl, 2, 7, str.tostring(current_n), text_color=#E6EDF3, text_size=size.tiny)
    table.cell(sqm_tbl, 3, 7, f_sqm_fmt(current_exp, "#.02") + "R | " + f_sqm_fmt(current_prob, "#.0") + "%", text_color=current_exp_col, text_size=size.tiny)

    table.cell(sqm_tbl, 0, 8, "النظام", text_color=#E6EDF3, bgcolor=#202A35, text_size=size.tiny)
    table.cell(sqm_tbl, 1, 8, "الصفقات", text_color=#8B98A5, bgcolor=#202A35, text_size=size.tiny)
    table.cell(sqm_tbl, 2, 8, "الفوز٪", text_color=#8B98A5, bgcolor=#202A35, text_size=size.tiny)
    table.cell(sqm_tbl, 3, 8, "التوقع", text_color=#8B98A5, bgcolor=#202A35, text_size=size.tiny)

    for r = 0 to sqm_regimes - 1
        string reg_label = r == 0 ? "TREND" : r == 1 ? "RANGE" : r == 2 ? "EXPANSION" : r == 3 ? "COMPRESSION" : "OTHER"

        int reg_n = array.get(sqm_reg_trades, r)
        int reg_w = array.get(sqm_reg_wins, r)
        float reg_sum = array.get(sqm_reg_sum_r, r)

        float reg_wr = reg_n > 0 ? reg_w / reg_n * 100.0 : na
        float reg_exp = reg_n > 0 ? reg_sum / reg_n : na

        color reg_wr_col = na(reg_wr) ? #8B98A5 : reg_wr >= 50.0 ? #26A69A : #EF5350
        color reg_exp_col = na(reg_exp) ? #8B98A5 : reg_exp > 0 ? #26A69A : reg_exp < 0 ? #EF5350 : #8B98A5

        int reg_row = 9 + r

        table.cell(sqm_tbl, 0, reg_row, reg_label, text_color=#E6EDF3, text_size=size.tiny)
        table.cell(sqm_tbl, 1, reg_row, str.tostring(reg_n), text_color=#8B98A5, text_size=size.tiny)
        table.cell(sqm_tbl, 2, reg_row, f_sqm_fmt(reg_wr, "#.0") + "%", text_color=reg_wr_col, text_size=size.tiny)
        table.cell(sqm_tbl, 3, reg_row, f_sqm_fmt(reg_exp, "#.02") + "R", text_color=reg_exp_col, text_size=size.tiny)

    string sqm_regime_current = regime_trend ? "TREND" : regime_range_ok ? "RANGE" : regime_expansion ? "EXPANSION" : regime_compression ? "COMPRESSION" : "WAIT"
    int sqm_mtf_align = (mtf1_state == "صاعد" ? 1 : mtf1_state == "هابط" ? -1 : 0) +
                         (mtf2_state == "صاعد" ? 1 : mtf2_state == "هابط" ? -1 : 0) +
                         (mtf3_state == "صاعد" ? 1 : mtf3_state == "هابط" ? -1 : 0)

    table.cell(sqm_tbl, 0, 14, "السياق", text_color=#8B98A5, text_size=size.tiny)
    table.cell(sqm_tbl, 1, 14, sqm_regime_current, text_color=#E6EDF3, text_size=size.tiny)
    table.cell(sqm_tbl, 2, 14, "الفريمات المتعددة " + str.tostring(sqm_mtf_align), text_color=sqm_mtf_align >= 2 or sqm_mtf_align <= -2 ? #26A69A : #F2C94C, text_size=size.tiny)
    table.cell(sqm_tbl, 3, 14, "رتبة ATR٪ " + str.tostring(nz(atr_rank, 0.0), "#.0"), text_color=nz(atr_rank, 0.0) >= atr_extreme_rank ? #EF5350 : #8B98A5, text_size=size.tiny)

// =============================================================================
// V13 STEP 8: STABILITY DASHBOARD
// NOTE: This must be placed INSIDE the stability engine block from Batch 3.
// Add the following at the END of the "if use_stability_engine" block,
// just before it closes.
// =============================================================================
// --- ADD THIS INSIDE THE STABILITY ENGINE BLOCK (end of Batch 3) ---

// =============================================================================
// V13 STEP 10: PRODUCTION READINESS DASHBOARD
// =============================================================================
if show_production_dashboard and barstate.islast and research_enabled
    var table prod_tbl = table.new(position.bottom_right, 2, 11, bgcolor=color.new(#111820, 92), border_width=1, border_color=#202A35)
    table.clear(prod_tbl, 0, 0, 1, 10)

    table.cell(prod_tbl, 0, 0, "🏁 بوابة الجاهزية", text_color=#E6EDF3, bgcolor=#202A35, text_size=size.tiny)
    table.cell(prod_tbl, 1, 0, prod_grade, text_color=prod_col, bgcolor=#202A35, text_size=size.small)

    table.cell(prod_tbl, 0, 1, "القرار", text_color=#8B98A5, text_size=size.tiny)
    table.cell(prod_tbl, 1, 1, prod_decision, text_color=prod_col, text_size=size.tiny)

    table.cell(prod_tbl, 0, 2, "الدرجة", text_color=#8B98A5, text_size=size.tiny)
    table.cell(prod_tbl, 1, 2, str.tostring(prod_score, "#.0") + "/100", text_color=prod_col, text_size=size.tiny)

    string prod_bar = ""
    int prod_blocks = int(math.round(prod_score / 10.0))
    for pb = 0 to 9
        prod_bar := prod_bar + (pb < prod_blocks ? "█" : "░")
    table.cell(prod_tbl, 0, 3, prod_bar, text_color=prod_col, text_size=size.tiny)
    table.cell(prod_tbl, 1, 3, "", text_color=color.new(color.gray, 100), text_size=size.tiny)

    table.cell(prod_tbl, 0, 4, "العينة", text_color=#8B98A5, text_size=size.tiny)
    table.cell(prod_tbl, 1, 4, str.tostring(prod_sample_score, "#.0") + "/20 | " + str.tostring(total_trades) + " trades", text_color=prod_sample_score >= 15 ? #26A69A : #F2C94C, text_size=size.tiny)

    table.cell(prod_tbl, 0, 5, "الميزة", text_color=#8B98A5, text_size=size.tiny)
    table.cell(prod_tbl, 1, 5, str.tostring(prod_edge_score, "#.0") + "/25 | PF " + str.tostring(nz(profit_factor, 0.0), "#.02"), text_color=prod_edge_score >= 18 ? #26A69A : #F2C94C, text_size=size.tiny)

    table.cell(prod_tbl, 0, 6, "المخاطر", text_color=#8B98A5, text_size=size.tiny)
    table.cell(prod_tbl, 1, 6, str.tostring(prod_risk_score, "#.0") + "/25 | DD " + str.tostring(nz(max_dd, 0.0), "#.00") + "R", text_color=prod_risk_score >= 18 ? #26A69A : #F2C94C, text_size=size.tiny)

    table.cell(prod_tbl, 0, 7, "الجودة", text_color=#8B98A5, text_size=size.tiny)
    table.cell(prod_tbl, 1, 7, str.tostring(prod_quality_score, "#.0") + "/15", text_color=prod_quality_score >= 10 ? #26A69A : #F2C94C, text_size=size.tiny)

    table.cell(prod_tbl, 0, 8, "الحداثة", text_color=#8B98A5, text_size=size.tiny)
    table.cell(prod_tbl, 1, 8, str.tostring(prod_recency_score, "#.0") + "/15", text_color=prod_recency_score >= 10 ? #26A69A : #F2C94C, text_size=size.tiny)

    table.cell(prod_tbl, 0, 9, "تحذير", text_color=#8B98A5, text_size=size.tiny)
    table.cell(prod_tbl, 1, 9, prod_warning, text_color=#F2C94C, text_size=size.tiny)

    table.cell(prod_tbl, 0, 10, "البروتوكول", text_color=#8B98A5, text_size=size.tiny)
    table.cell(prod_tbl, 1, 10, "خارج العينة → مونتِ كارلو → اختبار متدرج → مباشر", text_color=#5B8DEF, text_size=size.tiny)

// =============================================================================
// L33: EXPORT / ALERT OUTPUT
// =============================================================================
alertcondition(research_enabled and (hyper_bull_final or strong_bull_final or quick_bull_final), title="APEXv13 Research Buy Setup", message='{"engine":"APEXv13","event":"setup","direction":"LONG","symbol":"{{ticker}}","tf":"{{interval}}","price":"{{close}}"}')
alertcondition(research_enabled and (hyper_bear_final or strong_bear_final or quick_bear_final), title="APEXv13 Research Sell Setup", message='{"engine":"APEXv13","event":"setup","direction":"SHORT","symbol":"{{ticker}}","tf":"{{interval}}","price":"{{close}}"}')

// =============================================================================
// L17: VISUALS — CLEAN ZONE RENDERING
// =============================================================================
bgcolor(show_visuals and show_regime_bg and adx_trend ? c_regime_trend : na, title="Trend Regime")
bgcolor(show_visuals and show_regime_bg and regime_range_ok ? c_regime_range : na, title="Range Regime")
bgcolor(show_visuals and show_regime_bg and atr_extreme ? c_regime_extreme : na, title="Extreme Volatility")
bgcolor(show_visuals and show_killzone_bg and in_session_asia ? c_asia : na, title="Asia Killzone")
bgcolor(show_visuals and show_killzone_bg and in_session_london ? c_london : na, title="London Killzone")
bgcolor(show_visuals and show_killzone_bg and in_session_ny ? c_ny : na, title="NY Killzone")

// --- OTE Lines / Box ---
var line fib_line_top_obj = na
var line fib_line_bot_obj = na
var box fib_fill_box_obj = na
bool draw_fib_now = show_visuals and valid_range and fib_confirmed and fib_style != "None"
if draw_fib_now
    string line_style_value = fib_line_style == "Dashed" ? line.style_dashed : fib_line_style == "Dotted" ? line.style_dotted : line.style_solid
    float visual_top = trend_down ? bear_ote_top : bull_ote_top
    float visual_bot = trend_down ? bear_ote_bottom : bull_ote_bottom
    if not na(visual_top) and not na(visual_bot)
        if fib_style == "Lines"
            if na(fib_line_top_obj)
                fib_line_top_obj := line.new(bar_index, visual_top, bar_index + 1, visual_top, color=fib_line_color, width=fib_line_width, style=line_style_value, extend=extend.both)
            else
                line.set_y1(fib_line_top_obj, visual_top)
                line.set_y2(fib_line_top_obj, visual_top)
            if na(fib_line_bot_obj)
                fib_line_bot_obj := line.new(bar_index, visual_bot, bar_index + 1, visual_bot, color=fib_line_color, width=fib_line_width, style=line_style_value, extend=extend.both)
            else
                line.set_y1(fib_line_bot_obj, visual_bot)
                line.set_y2(fib_line_bot_obj, visual_bot)
            if na(fib_fill_box_obj)
                fib_fill_box_obj := box.new(bar_index, visual_top, bar_index + 1, visual_bot, bgcolor=fib_fill_color, border_color=color.new(color.gray, 100), border_width=0, extend=extend.both)
            else
                box.set_top(fib_fill_box_obj, visual_top)
                box.set_bottom(fib_fill_box_obj, visual_bot)
        else
            int fib_left = math.max(0, bar_index - 50)
            if na(fib_fill_box_obj)
                fib_fill_box_obj := box.new(fib_left, visual_top, bar_index + 50, visual_bot, bgcolor=color.new(fib_line_color, 85), border_color=fib_line_color, extend=extend.both)
            else
                box.set_top(fib_fill_box_obj, visual_top)
                box.set_bottom(fib_fill_box_obj, visual_bot)
else
    if not na(fib_line_top_obj)
        line.set_color(fib_line_top_obj, color.new(color.gray, 100))
    if not na(fib_line_bot_obj)
        line.set_color(fib_line_bot_obj, color.new(color.gray, 100))
    if not na(fib_fill_box_obj)
        box.set_bgcolor(fib_fill_box_obj, color.new(color.gray, 100))
        box.set_border_color(fib_fill_box_obj, color.new(color.gray, 100))

// --- OB Boxes ---
var box[] ob_bull_boxes = array.new_box(0)
var box[] ob_bear_boxes = array.new_box(0)
var int max_ob_boxes = 15
if show_visuals and ob_mitigation and confirmed
    int ob_left = math.max(0, bar_index - 1)
    int ob_bull_drawn = 0
    if ob_bull_count > 0
        for ob_draw_i = 0 to math.min(ob_process_limit, ob_bull_count) - 1
            if array.get(ob_bull_valids, ob_draw_i)
                int age_d = bar_index - array.get(ob_bull_bars, ob_draw_i)
                int score_d = array.get(ob_bull_scores, ob_draw_i)
                bool is_primary = score_d >= 70 and age_d <= 8
                bool is_secondary = score_d >= 55 and age_d <= 16
                if is_primary or is_secondary
                    int transp_d = is_primary ? 78 : 88
                    color border_col = is_primary ? c_bull : color.new(c_bull, 60)
                    color bg_col = is_primary ? color.new(c_bull, transp_d) : color.new(c_bull, transp_d + 10)
                    box bx = box.new(ob_left, array.get(ob_bull_tops, ob_draw_i), bar_index, array.get(ob_bull_bots, ob_draw_i), bgcolor=bg_col, border_color=border_col, border_width=1, extend=extend.right)
                    array.push(ob_bull_boxes, bx)
                    ob_bull_drawn += 1
                    if ob_bull_drawn >= 3
                        break
    int ob_bull_box_size = array.size(ob_bull_boxes)
    if ob_bull_box_size > max_ob_boxes
        int ob_bull_del_count = ob_bull_box_size - max_ob_boxes
        for del_i = 0 to ob_bull_del_count - 1
            box.delete(array.shift(ob_bull_boxes))
    int ob_bear_drawn = 0
    if ob_bear_count > 0
        for ob_draw_i = 0 to math.min(ob_process_limit, ob_bear_count) - 1
            if array.get(ob_bear_valids, ob_draw_i)
                int age_d = bar_index - array.get(ob_bear_bars, ob_draw_i)
                int score_d = array.get(ob_bear_scores, ob_draw_i)
                bool is_primary = score_d >= 70 and age_d <= 8
                bool is_secondary = score_d >= 55 and age_d <= 16
                if is_primary or is_secondary
                    int transp_d = is_primary ? 78 : 88
                    color border_col = is_primary ? c_bear : color.new(c_bear, 60)
                    color bg_col = is_primary ? color.new(c_bear, transp_d) : color.new(c_bear, transp_d + 10)
                    box bx = box.new(ob_left, array.get(ob_bear_tops, ob_draw_i), bar_index, array.get(ob_bear_bots, ob_draw_i), bgcolor=bg_col, border_color=border_col, border_width=1, extend=extend.right)
                    array.push(ob_bear_boxes, bx)
                    ob_bear_drawn += 1
                    if ob_bear_drawn >= 3
                        break
    int ob_bear_box_size = array.size(ob_bear_boxes)
    if ob_bear_box_size > max_ob_boxes
        int ob_bear_del_count = ob_bear_box_size - max_ob_boxes
        for del_i = 0 to ob_bear_del_count - 1
            box.delete(array.shift(ob_bear_boxes))

// --- FVG Boxes + Inversion ---
var box[] fvg_bull_boxes = array.new_box(0)
var box[] fvg_bear_boxes = array.new_box(0)
var box[] inv_bull_boxes = array.new_box(0)
var box[] inv_bear_boxes = array.new_box(0)
var int max_fvg_boxes = 12
if show_visuals and confirmed
    int zone_left = math.max(0, bar_index - 2)
    int fvg_bull_drawn = 0
    if fvg_bull_count > 0
        for fvg_draw_i = 0 to math.min(fvg_process_limit, fvg_bull_count) - 1
            if array.get(fvg_bull_valids, fvg_draw_i)
                int age_d = bar_index - array.get(fvg_bull_bars, fvg_draw_i)
                if age_d <= 10
                    box bx = box.new(zone_left, array.get(fvg_bull_tops, fvg_draw_i), bar_index, array.get(fvg_bull_bots, fvg_draw_i), bgcolor=color.new(c_bull, 82), border_color=color.new(c_bull, 50), border_width=1, extend=extend.right)
                    array.push(fvg_bull_boxes, bx)
                    fvg_bull_drawn += 1
                    if fvg_bull_drawn >= 2
                        break
    int fvg_bull_box_size = array.size(fvg_bull_boxes)
    if fvg_bull_box_size > max_fvg_boxes
        int fvg_bull_del_count = fvg_bull_box_size - max_fvg_boxes
        for del_i = 0 to fvg_bull_del_count - 1
            box.delete(array.shift(fvg_bull_boxes))
    int fvg_bear_drawn = 0
    if fvg_bear_count > 0
        for fvg_draw_i = 0 to math.min(fvg_process_limit, fvg_bear_count) - 1
            if array.get(fvg_bear_valids, fvg_draw_i)
                int age_d = bar_index - array.get(fvg_bear_bars, fvg_draw_i)
                if age_d <= 10
                    box bx = box.new(zone_left, array.get(fvg_bear_tops, fvg_draw_i), bar_index, array.get(fvg_bear_bots, fvg_draw_i), bgcolor=color.new(c_bear, 82), border_color=color.new(c_bear, 50), border_width=1, extend=extend.right)
                    array.push(fvg_bear_boxes, bx)
                    fvg_bear_drawn += 1
                    if fvg_bear_drawn >= 2
                        break
    int fvg_bear_box_size = array.size(fvg_bear_boxes)
    if fvg_bear_box_size > max_fvg_boxes
        int fvg_bear_del_count = fvg_bear_box_size - max_fvg_boxes
        for del_i = 0 to fvg_bear_del_count - 1
            box.delete(array.shift(fvg_bear_boxes))
    int inv_bull_count = array.size(inv_bull_tops)
    int inv_bear_count = array.size(inv_bear_tops)
    if inv_bull_count > 0
        for inv_draw_i = 0 to math.min(2, inv_bull_count) - 1
            if array.get(inv_bull_valids, inv_draw_i)
                int age_d = bar_index - array.get(inv_bull_bars, inv_draw_i)
                if age_d <= 12
                    box bx = box.new(zone_left, array.get(inv_bull_tops, inv_draw_i), bar_index, array.get(inv_bull_bots, inv_draw_i), bgcolor=color.new(c_warning, 88), border_color=color.new(c_warning, 60), border_width=1, border_style=line.style_dashed, extend=extend.right)
                    array.push(inv_bull_boxes, bx)
    int inv_bull_box_size = array.size(inv_bull_boxes)
    if inv_bull_box_size > 6
        int inv_bull_del_count = inv_bull_box_size - 6
        for del_i = 0 to inv_bull_del_count - 1
            box.delete(array.shift(inv_bull_boxes))
    if inv_bear_count > 0
        for inv_draw_i = 0 to math.min(2, inv_bear_count) - 1
            if array.get(inv_bear_valids, inv_draw_i)
                int age_d = bar_index - array.get(inv_bear_bars, inv_draw_i)
                if age_d <= 12
                    box bx = box.new(zone_left, array.get(inv_bear_tops, inv_draw_i), bar_index, array.get(inv_bear_bots, inv_draw_i), bgcolor=color.new(c_warning, 88), border_color=color.new(c_warning, 60), border_width=1, border_style=line.style_dashed, extend=extend.right)
                    array.push(inv_bear_boxes, bx)
    int inv_bear_box_size = array.size(inv_bear_boxes)
    if inv_bear_box_size > 6
        int inv_bear_del_count = inv_bear_box_size - 6
        for del_i = 0 to inv_bear_del_count - 1
            box.delete(array.shift(inv_bear_boxes))

// =============================================================================
// LABELS LIFECYCLE MANAGEMENT
// =============================================================================
var label[] signal_labels = array.new_label(0)
var int max_signal_labels = 30
add_signal_label(label lbl) =>
    array.push(signal_labels, lbl)
    if array.size(signal_labels) > max_signal_labels
        label.delete(array.shift(signal_labels))
    true

// --- Debug marks ---
plotshape(show_debug_marks and bos_bull, title="BOS Bull", text="BOS", style=shape.triangleup, location=location.belowbar, color=c_bull, size=size.tiny)
plotshape(show_debug_marks and bos_bear, title="BOS Bear", text="BOS", style=shape.triangledown, location=location.abovebar, color=c_bear, size=size.tiny)
plotshape(show_debug_marks and liq_sweep_low, title="Sweep Low", text="SW", style=shape.circle, location=location.belowbar, color=c_warning, size=size.tiny)
plotshape(show_debug_marks and liq_sweep_high, title="Sweep High", text="SW", style=shape.circle, location=location.abovebar, color=c_warning, size=size.tiny)
plotshape(show_debug_marks and liquidity_void_bull, title="Void Bull", text="VOID", style=shape.diamond, location=location.belowbar, color=color.new(#9C27B0, 0), size=size.tiny)
plotshape(show_debug_marks and liquidity_void_bear, title="Void Bear", text="VOID", style=shape.diamond, location=location.abovebar, color=color.new(#9C27B0, 0), size=size.tiny)

// --- Watch signals ---
plotshape(watch_bull, title="راقب شراء", text="راقب", style=shape.circle, location=location.belowbar, color=color.new(c_bull, 20), textcolor=c_bull, size=size.tiny)
plotshape(watch_bear, title="Watch Bear", text="راقب", style=shape.circle, location=location.abovebar, color=color.new(c_bear, 20), textcolor=c_bear, size=size.tiny)
plotshape(realtime_bull_final, title="RT Buy", text="فوري", style=shape.labelup, location=location.belowbar, color=color.new(c_warning, 0), textcolor=color.black, size=size.tiny)
plotshape(realtime_bear_final, title="RT Sell", text="فوري", style=shape.labeldown, location=location.abovebar, color=color.new(c_warning, 0), textcolor=color.black, size=size.tiny)

// --- Signal labels ---
if quick_bull_final
    add_signal_label(label.new(bar_index, low - safe_atr * 0.35, "Q " + str.tostring(bull_score), color=c_quick, textcolor=color.white, style=label.style_label_up, size=size.small))
if quick_bear_final
    add_signal_label(label.new(bar_index, high + safe_atr * 0.35, "Q " + str.tostring(bear_score), color=c_quick, textcolor=color.white, style=label.style_label_down, size=size.small))
if strong_bull_final
    add_signal_label(label.new(bar_index, low - safe_atr * 0.45, "S " + str.tostring(bull_score), color=c_strong, textcolor=color.white, style=label.style_label_up, size=size.small))
if strong_bear_final
    add_signal_label(label.new(bar_index, high + safe_atr * 0.45, "S " + str.tostring(bear_score), color=c_strong, textcolor=color.white, style=label.style_label_down, size=size.small))
if hyper_bull_final
    add_signal_label(label.new(bar_index, low - safe_atr * 0.55, "H " + str.tostring(bull_score), color=c_hyper, textcolor=color.white, style=label.style_label_up, size=size.small))
if hyper_bear_final
    add_signal_label(label.new(bar_index, high + safe_atr * 0.55, "H " + str.tostring(bear_score), color=c_hyper, textcolor=color.white, style=label.style_label_down, size=size.small))
if show_blocked_signals
    if quick_bull_blocked or strong_bull_blocked or hyper_bull_blocked
        add_signal_label(label.new(bar_index, low - safe_atr * 0.25, "🚫", color=color.gray, textcolor=color.white, style=label.style_label_up, size=size.tiny))
    if quick_bear_blocked or strong_bear_blocked or hyper_bear_blocked
        add_signal_label(label.new(bar_index, high + safe_atr * 0.25, "🚫", color=color.gray, textcolor=color.white, style=label.style_label_down, size=size.tiny))
        // =============================================================================
// L18: EXPLAINABLE DASHBOARD — PREMIUM UI
// (V13: Step 4 Regime Matrix + Step 6 MTF Architecture integrated)
// =============================================================================
if show_panel and barstate.islast
    var table premium = table.new(position.top_right, 2, 16, bgcolor=color.new(#111820, 94), border_width=1, border_color=#202A35)
    table.clear(premium, 0, 0, 1, 15)
    table.cell(premium, 0, 0, "عقل APEX", text_color=#E6EDF3, text_size=size.normal, bgcolor=#202A35)
    string header_market = syminfo.ticker + " · " + timeframe.period
    table.cell(premium, 1, 0, header_market, text_color=#8B98A5, text_size=size.small, bgcolor=#202A35)
    string live_status = "● LIVE"
    color live_col = #26A69A

    // V13 STEP 4: Use Advanced Regime Matrix instead of old regime logic
    string regime_badge = regime_matrix_text
    color regime_badge_col = regime_matrix_color

    table.cell(premium, 0, 1, live_status, text_color=live_col, text_size=size.small, bgcolor=#111820)
    table.cell(premium, 1, 1, regime_badge, text_color=regime_badge_col, text_size=size.small, bgcolor=#111820)

    // V13 STEP 4: Row 2 — Regime Quality (was empty separator)
    table.cell(premium, 0, 2, "النظام", text_color=#8B98A5, text_size=size.tiny, bgcolor=#111820)
    table.cell(premium, 1, 2, regime_matrix_text + " | Q " + str.tostring(regime_matrix_quality * 100.0, "#.0") + "%", text_color=regime_matrix_color, text_size=size.tiny, bgcolor=#111820)

    bool bull_dominant = bull_score >= bear_score and bull_score >= 35
    bool bear_dominant = bear_score > bull_score and bear_score >= 35
    bool neutral = not bull_dominant and not bear_dominant
    string decision_text = neutral ? "WAIT" : bull_dominant ? "BUY" : "SELL"
    color decision_color = neutral ? #8B98A5 : bull_dominant ? #26A69A : #EF5350
    int decision_score = neutral ? 0 : bull_dominant ? bull_score : bear_score
    string decision_label = neutral ? "NO VALID SETUP" : (decision_score >= profile_hyper_score_min ? "فائق" : decision_score >= profile_strong_score_min ? "قوي" : decision_score >= profile_quick_score_min ? "سريع" : "WEAK")
    color decision_label_col = neutral ? #8B98A5 : decision_score >= profile_hyper_score_min ? #9C27B0 : decision_score >= profile_strong_score_min ? #F44336 : decision_score >= profile_quick_score_min ? #00BCD4 : #8B98A5

    table.cell(premium, 0, 3, "⚡ DECISION", text_color=#8B98A5, text_size=size.small, bgcolor=#111820)
    table.cell(premium, 1, 3, decision_text, text_color=decision_color, text_size=size.normal, bgcolor=#111820)
    table.cell(premium, 0, 4, "الدرجة", text_color=#8B98A5, text_size=size.tiny, bgcolor=#111820)
    table.cell(premium, 1, 4, str.tostring(decision_score) + "/100", text_color=decision_color, text_size=size.small, bgcolor=#111820)

    string bar_len = ""
    int bar_count = int(math.round(decision_score / 5.0))
    if bar_count > 0
        for b = 0 to math.min(bar_count - 1, 19)
            bar_len := bar_len + "█"
    if bar_count < 20
        for b = math.max(bar_count, 0) to 19
            bar_len := bar_len + "░"
    table.cell(premium, 0, 5, bar_len, text_color=decision_color, text_size=size.tiny, bgcolor=#111820)
    table.cell(premium, 1, 5, "", text_color=color.new(color.gray, 100), bgcolor=#111820)

    table.cell(premium, 0, 6, "الحالة", text_color=#8B98A5, text_size=size.tiny, bgcolor=#111820)
    table.cell(premium, 1, 6, decision_label, text_color=decision_label_col, text_size=size.small, bgcolor=#111820)

    table.cell(premium, 0, 7, "", text_color=#202A35, bgcolor=#202A35)
    table.cell(premium, 1, 7, "", text_color=#202A35, bgcolor=#202A35)

    string htf_text = not use_htf_trend ? "OFF" : htf_power_bull ? "POWER BULL" : htf_power_bear ? "POWER BEAR" : htf_trend_up ? "BULLISH" : htf_trend_down ? "BEARISH" : "N/A"
    color htf_col = not use_htf_trend ? #8B98A5 : htf_power_bull or htf_trend_up ? #26A69A : htf_power_bear or htf_trend_down ? #EF5350 : #8B98A5
    table.cell(premium, 0, 8, "الفريم الأعلى", text_color=#8B98A5, text_size=size.tiny, bgcolor=#111820)
    table.cell(premium, 1, 8, htf_text, text_color=htf_col, text_size=size.small, bgcolor=#111820)

    string session_text = in_session_london ? "LONDON" : in_session_ny ? "NEW YORK" : in_session_asia ? "ASIA" : "NONE"
    color session_col = in_session_london ? #FF9800 : in_session_ny ? #4CAF50 : in_session_asia ? #2196F3 : #8B98A5
    string killzone_text = london_sweep_bull ? "SWEEP UP" : london_sweep_bear ? "SWEEP DOWN" : ny_expansion_bull ? "NY BULL" : ny_expansion_bear ? "NY BEAR" : "-"
    table.cell(premium, 0, 9, "الجلسة", text_color=#8B98A5, text_size=size.tiny, bgcolor=#111820)
    table.cell(premium, 1, 9, session_text + (killzone_text != "-" ? " | " + killzone_text : ""), text_color=session_col, text_size=size.small, bgcolor=#111820)

    string rr_text = not use_risk_filter ? "OFF" : (not na(bull_rr) and bull_score >= bear_score ? str.tostring(bull_rr, "#.0") + "R" : not na(bear_rr) and bear_score > bull_score ? str.tostring(bear_rr, "#.0") + "R" : "-")
    color rr_col = rr_text == "OFF" ? #8B98A5 : (rr_text != "-" and (bull_rr_ok or bear_rr_ok) ? #26A69A : #EF5350)
    table.cell(premium, 0, 10, "العائد/المخاطرة", text_color=#8B98A5, text_size=size.tiny, bgcolor=#111820)
    table.cell(premium, 1, 10, rr_text, text_color=rr_col, text_size=size.small, bgcolor=#111820)

    string mom_text = momentum_bull ? "صاعد" : momentum_bear ? "هابط" : "NEUTRAL"
    color mom_col = momentum_bull ? #26A69A : momentum_bear ? #EF5350 : #8B98A5
    table.cell(premium, 0, 11, "الزخم", text_color=#8B98A5, text_size=size.tiny, bgcolor=#111820)
    table.cell(premium, 1, 11, mom_text + " " + str.tostring(momentum_strength * 100, "#.0") + "%", text_color=mom_col, text_size=size.small, bgcolor=#111820)

    // V13 STEP 6: Row 12 — MTF Architecture (was empty separator)
    string mtf_arch_text = "MTF: " + str.tostring(mtf_bull_alignment) + "↑ / " + str.tostring(mtf_bear_alignment) + "↓"
    color mtf_arch_col = mtf_bull_aligned ? #26A69A : mtf_bear_aligned ? #EF5350 : #F2C94C
    table.cell(premium, 0, 12, "بنية الفريمات المتعددة", text_color=#8B98A5, text_size=size.tiny, bgcolor=#111820)
    table.cell(premium, 1, 12, mtf_arch_text, text_color=mtf_arch_col, text_size=size.tiny, bgcolor=#111820)

    table.cell(premium, 0, 13, "التدفق", text_color=#8B98A5, text_size=size.tiny, bgcolor=#111820)
    string flow_text = ""
    color flow_col = #8B98A5
    if bull_dominant
        flow_text := _sweep_ok_bull ? "SWEEP✓ " : "SWEEP✗ "
        flow_text := flow_text + (bos_ok_bull ? "BOS✓ " : "BOS✗ ")
        flow_text := flow_text + (_ob_ok_bull ? "OB✓ " : _fvg_ok_bull ? "FVG✓ " : "ZONE✗ ")
        flow_text := flow_text + (in_bull_ote ? "OTE✓" : "OTE✗")
        flow_col := _sweep_ok_bull and bos_ok_bull and (_ob_ok_bull or _fvg_ok_bull) ? #26A69A : #F2C94C
    else if bear_dominant
        flow_text := _sweep_ok_bear ? "SWEEP✓ " : "SWEEP✗ "
        flow_text := flow_text + (bos_ok_bear ? "BOS✓ " : "BOS✗ ")
        flow_text := flow_text + (_ob_ok_bear ? "OB✓ " : _fvg_ok_bear ? "FVG✓ " : "ZONE✗ ")
        flow_text := flow_text + (in_bear_ote ? "OTE✓" : "OTE✗")
        flow_col := _sweep_ok_bear and bos_ok_bear and (_ob_ok_bear or _fvg_ok_bear) ? #EF5350 : #F2C94C
    else
        flow_text := "WAITING FOR SETUP"
        flow_col := #8B98A5
    table.cell(premium, 0, 14, flow_text, text_color=flow_col, text_size=size.tiny, bgcolor=#111820)
    table.cell(premium, 1, 14, "", text_color=color.new(color.gray, 100), bgcolor=#111820)

    if use_mtf_dashboard
        string mtf1_icon = mtf1_state == "صاعد" ? "🟢" : mtf1_state == "هابط" ? "🔴" : "⚪"
        string mtf2_icon = mtf2_state == "صاعد" ? "🟢" : mtf2_state == "هابط" ? "🔴" : "⚪"
        string mtf3_icon = mtf3_state == "صاعد" ? "🟢" : mtf3_state == "هابط" ? "🔴" : "⚪"
        int mtf_alignment = (mtf1_state == "صاعد" ? 1 : mtf1_state == "هابط" ? -1 : 0) +
                             (mtf2_state == "صاعد" ? 1 : mtf2_state == "هابط" ? -1 : 0) +
                             (mtf3_state == "صاعد" ? 1 : mtf3_state == "هابط" ? -1 : 0)
        string align_text = mtf_alignment >= 2 ? "✓ ALIGNED" : mtf_alignment <= -2 ? "✓ ALIGNED" : "MIXED"
        color align_col = mtf_alignment >= 2 or mtf_alignment <= -2 ? #26A69A : #F2C94C
        table.cell(premium, 0, 15, mtf_tf1 + " " + mtf1_icon + "  " + mtf_tf2 + " " + mtf2_icon + "  " + mtf_tf3 + " " + mtf3_icon, text_color=#8B98A5, text_size=size.tiny, bgcolor=#111820)
        table.cell(premium, 1, 15, align_text, text_color=align_col, text_size=size.tiny, bgcolor=#111820)

// =============================================================================
// L19: ALERTS (Updated to APEXv13)
// =============================================================================
alertcondition(quick_bull_final, title="APEXv13 Quick Buy", message='{"engine":"APEXv13","signal":"quick_buy","symbol":"{{ticker}}","tf":"{{interval}}","price":"{{close}}"}')
alertcondition(quick_bear_final, title="APEXv13 Quick Sell", message='{"engine":"APEXv13","signal":"quick_sell","symbol":"{{ticker}}","tf":"{{interval}}","price":"{{close}}"}')
alertcondition(strong_bull_final, title="APEXv13 Strong Buy", message='{"engine":"APEXv13","signal":"strong_buy","symbol":"{{ticker}}","tf":"{{interval}}","price":"{{close}}"}')
alertcondition(strong_bear_final, title="APEXv13 Strong Sell", message='{"engine":"APEXv13","signal":"strong_sell","symbol":"{{ticker}}","tf":"{{interval}}","price":"{{close}}"}')
alertcondition(hyper_bull_final, title="APEXv13 Hyper Buy", message='{"engine":"APEXv13","signal":"hyper_buy","symbol":"{{ticker}}","tf":"{{interval}}","price":"{{close}}"}')
alertcondition(hyper_bear_final, title="APEXv13 Hyper Sell", message='{"engine":"APEXv13","signal":"hyper_sell","symbol":"{{ticker}}","tf":"{{interval}}","price":"{{close}}"}')
alertcondition(realtime_alerts and realtime_bull_final, title="APEXv13 Realtime Buy", message='{"engine":"APEXv13","signal":"realtime_early_buy","symbol":"{{ticker}}","tf":"{{interval}}","price":"{{close}}"}')
alertcondition(realtime_alerts and realtime_bear_final, title="APEXv13 Realtime Sell", message='{"engine":"APEXv13","signal":"realtime_early_sell","symbol":"{{ticker}}","tf":"{{interval}}","price":"{{close}}"}')
alertcondition(show_decision_stage and watch_bull, title="APEXv13 Watch Buy", message='{"engine":"APEXv13","signal":"watch_buy","symbol":"{{ticker}}","tf":"{{interval}}","price":"{{close}}"}')
alertcondition(show_decision_stage and watch_bear, title="APEXv13 Watch Sell", message='{"engine":"APEXv13","signal":"watch_sell","symbol":"{{ticker}}","tf":"{{interval}}","price":"{{close}}"}')

// =============================================================================
// EQUITY CURVE PLOT
// =============================================================================
plot(show_equity_curve and research_enabled ? equity_r : na, title="الرصيد (R)", color=color.new(#26A69A, 0), linewidth=2, display=display.data_window)
