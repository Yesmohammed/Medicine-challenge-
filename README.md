# Medicine-challenge-
Challenge your ability to invent kidney strock and blood stick by machine 

---

### 💻 الجزء الثاني: النموذج الأولي لواجهة المحاكاة (GUI) - كود Python جاهز للتشغيل
احفظ الكود التالي باسم `gui_controller.py` في مجلد `/gui`، وستظهر لك نافذة تحكم تفاعلية تحاكي مزج المعايير والاستجابة الحيوية.

```python
import tkinter as tk
from tkinter import ttk, scrolledtext
import random
import threading
import time

class MedicalVesselGUI:
    def __init__(self, root):
        self.root = root
        self.root.title("المركبة الطبية الذكية - محاكاة التحكم (v1.0)")
        self.root.geometry("1000x700")
        self.root.configure(bg='#f0f4f8')

        # متغيرات حالة المريض (محاكاة)
        self.heart_rate = 72
        self.blood_pressure = 120
        self.temp = 36.8
        self.fragmentation = 0  # نسبة التفتيت

        # متغيرات التحكم (المُزج)
        self.power = tk.DoubleVar(value=30.0)
        self.frequency = tk.DoubleVar(value=100.0)
        self.depth = tk.DoubleVar(value=5.0)
        self.water_pressure = tk.DoubleVar(value=1.2)
        self.air_pressure = tk.DoubleVar(value=35.0)

        self.is_treating = False
        self.is_emergency = False

        # بناء الواجهة
        self.create_widgets()
        self.update_status()

    def create_widgets(self):
        # الإطار الرئيسي
        main_frame = ttk.Frame(self.root, padding="10")
        main_frame.pack(fill=tk.BOTH, expand=True)

        # ------ العمود الأيمن: التحكم في المعايير (المزج) ------
        control_frame = ttk.LabelFrame(main_frame, text="🧪 معايير المزج الديناميكي", padding="10")
        control_frame.pack(side=tk.LEFT, fill=tk.BOTH, expand=True, padx=5)

        # شرائح التحكم
        sliders = [
            ("⚡ الطاقة (ميجاباسكال)", self.power, 10, 80, "%.1f"),
            ("🎵 التردد (كيلوهرتز)", self.frequency, 20, 500, "%.0f"),
            ("📏 العمق (سم)", self.depth, 2, 12, "%.1f"),
            ("💧 ضغط الماء (بار)", self.water_pressure, 0.5, 2.0, "%.2f"),
            ("💨 ضغط الهواء (ملمزئبقي)", self.air_pressure, 10, 50, "%.0f")
        ]

        self.slider_vars = {}
        for i, (label, var, min_val, max_val, fmt) in enumerate(sliders):
            frame = ttk.Frame(control_frame)
            frame.pack(fill=tk.X, pady=5)

            ttk.Label(frame, text=label, font=('Arial', 10, 'bold')).pack(anchor='w')
            slider = ttk.Scale(frame, from_=min_val, to=max_val, orient='horizontal', variable=var, length=300)
            slider.pack(side=tk.LEFT, fill=tk.X, expand=True)

            val_label = ttk.Label(frame, text=f"{fmt % var.get()}", font=('Arial', 10))
            val_label.pack(side=tk.RIGHT, padx=10)
            # حفظ مرجع لتحديث النص عند تغيير القيمة
            self.slider_vars[var] = (val_label, fmt)

            # تحديث التسميات ديناميكياً
            var.trace('w', lambda *args, v=var, lbl=val_label, f=fmt: lbl.config(text=f % v.get()))

        # ------ العمود الأيسر: حالة المريض والتغذية الراجعة ------
        vitals_frame = ttk.LabelFrame(main_frame, text="🫀 المؤشرات الحيوية & التفتيت", padding="10")
        vitals_frame.pack(side=tk.RIGHT, fill=tk.BOTH, expand=True, padx=5)

        self.vitals_labels = {}
        vitals = [
            ("❤️ النبض (BPM)", f"{self.heart_rate}"),
            ("🩸 الضغط الانقباضي", f"{self.blood_pressure}"),
            ("🌡️ الحرارة (°C)", f"{self.temp}"),
            ("🧊 نسبة التفتيت (%)", f"{self.fragmentation}")
        ]
        for i, (text, val) in enumerate(vitals):
            frame = ttk.Frame(vitals_frame)
            frame.pack(fill=tk.X, pady=8)
            ttk.Label(frame, text=text, font=('Arial', 12)).pack(side=tk.LEFT)
            lbl = ttk.Label(frame, text=val, font=('Arial', 14, 'bold'), foreground='#2c3e50')
            lbl.pack(side=tk.RIGHT)
            self.vitals_labels[text] = lbl

        # مربع السجل (Log)
        log_frame = ttk.LabelFrame(vitals_frame, text="📋 سجل العلاج", padding="5")
        log_frame.pack(fill=tk.BOTH, expand=True, pady=10)

        self.log_text = scrolledtext.ScrolledText(log_frame, height=8, font=('Arial', 10), wrap=tk.WORD)
        self.log_text.pack(fill=tk.BOTH, expand=True)
        self.log_text.insert(tk.END, "🟢 الجهاز جاهز. انتظر بدء المعايرة...\n")
        self.log_text.see(tk.END)

        # ------ أزرار التحكم السفلى ------
        btn_frame = ttk.Frame(main_frame)
        btn_frame.pack(side=tk.BOTTOM, fill=tk.X, pady=10)

        ttk.Button(btn_frame, text="🚀 بدء المحاكاة", command=self.start_treatment, width=20).pack(side=tk.LEFT, padx=5)
        self.emergency_btn = ttk.Button(btn_frame, text="🛑 إيقاف طوارئ (أحمر)", command=self.emergency_stop, width=25)
        self.emergency_btn.pack(side=tk.LEFT, padx=5)

        ttk.Button(btn_frame, text="🧹 معايرة ذاتية", command=self.run_calibration, width=20).pack(side=tk.LEFT, padx=5)
        ttk.Button(btn_frame, text="❌ إغلاق", command=self.root.quit, width=15).pack(side=tk.RIGHT, padx=5)

    # ------------------- منطق المحاكاة -------------------
    def log(self, msg):
        self.log_text.insert(tk.END, f"{msg}\n")
        self.log_text.see(tk.END)

    def run_calibration(self):
        self.log("🧪 بدء المعايرة الذاتية...")
        self.root.update()
        time.sleep(1)
        # محاكاة فحص الضغوط والترددات
        self.log("✅ ضغط الماء: ثابت عند 1.2 بار.")
        self.log("✅ ضغط الهواء: ثابت عند 35 ملمزئبقي.")
        self.log("✅ فحص التردد: استجابة طبيعية (Q-Factor = 12).")
        self.log("✅ المعايرة ناجحة. الجهاز جاهز للعلاج.")

    def start_treatment(self):
        if self.is_treating:
            self.log("⚠️ العلاج قيد التشغيل بالفعل.")
            return
        if self.is_emergency:
            self.log("⛔ تم الضغط على الطوارئ. أعد تشغيل المحاكاة أولاً.")
            return

        self.is_treating = True
        self.log(f"💥 بدء العلاج...")
        self.log(f"   المعايير: طاقة={self.power.get():.1f}, تردد={self.frequency.get():.0f} كيلوهرتز")

        # محاكاة في خيط منفصل حتى لا تتجمد الواجهة
        thread = threading.Thread(target=self.simulate_treatment)
        thread.daemon = True
        thread.start()

    def simulate_treatment(self):
        for i in range(10):  # 10 خطوات محاكاة
            if self.is_emergency or not self.is_treating:
                break

            # محاكاة تحسن التفتيت بناءً على الطاقة والتردد
            increment = (self.power.get() * 0.05) + (self.frequency.get() * 0.001)
            self.fragmentation = min(100, self.fragmentation + increment * random.uniform(0.8, 1.2))

            # محاكاة استجابة حيوية (تغير طفيف في النبض والضغط)
            self.heart_rate = 72 + random.randint(-3, 5)
            self.blood_pressure = 120 + random.randint(-5, 8)
            self.temp = 36.8 + random.uniform(0, 0.3)

            # تحديث الواجهة
            self.update_vitals()
            self.log(f"🔹 نبضة {i+1}: التفتيت = {self.fragmentation:.1f}% | حرارة = {self.temp:.1f}°")

            # محاكاة خروج شظية (مرئي)
            if random.random() > 0.7:
                self.log("✅ ✅ كاميرا المخرج: رصدت شظية خارجة! (قطر 0.8 مم)")

            time.sleep(1.5)

        if self.fragmentation >= 95 and not self.is_emergency:
            self.log("🎉 🎉 اكتمل العلاج! نسبة التفتيت 95%+.")
        self.is_treating = False

    def emergency_stop(self):
        if self.is_emergency:
            return
        self.is_emergency = True
        self.is_treating = False
        self.log("🚨 🚨 تم الضغط على إيقاف الطوارئ! إيقاف جميع المحركات والموجات.")
        self.log("🛡️ تم تفريغ الهواء من الكفن، وتحرير المكابح.")
        self.emergency_btn.config(style='danger.TButton')
        self.root.update()

        # محاكاة إعادة الضبط بعد 3 ثوانٍ كإجراء أمان
        def reset_emergency():
            time.sleep(3)
            self.is_emergency = False
            self.log("🔄 إعادة ضبط الطوارئ. يمكن بدء معايرة جديدة.")
            self.emergency_btn.config(style='TButton')
        threading.Thread(target=reset_emergency, daemon=True).start()

    def update_vitals(self):
        try:
            self.vitals_labels["❤️ النبض (BPM)"].config(text=f"{self.heart_rate}")
            self.vitals_labels["🩸 الضغط الانقباضي"].config(text=f"{self.blood_pressure}")
            self.vitals_labels["🌡️ الحرارة (°C)"].config(text=f"{self.temp:.1f}")
            self.vitals_labels["🧊 نسبة التفتيت (%)"].config(text=f"{self.fragmentation:.1f}")
        except:
            pass

    def update_status(self):
        # تحديث دوري للمؤشرات حتى أثناء الانتظار
        if not self.is_treating and not self.is_emergency:
            self.fragmentation = max(0, self.fragmentation - 0.1)
            self.update_vitals()
        self.root.after(1000, self.update_status)

# تشغيل التطبيق
if __name__ == "__main__":
    root = tk.Tk()
    # نمط أزرار الطوارئ (جعلها حمراء)
    style = ttk.Style()
    style.configure('danger.TButton', foreground='white', background='red', font=('Arial', 10, 'bold'))
    app = MedicalVesselGUI(root)
    root.mainloop()
