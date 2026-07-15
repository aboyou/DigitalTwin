برای اتصال سامانه به یک دیتابیس جدید، دو حالت دارید.

## حالت اول: برای دیتابیس جدید، خروجی تحلیل شبکه دارید

فرض کنیم فایل‌های جدید شما این‌ها هستند:

```text
D:\new_data\articles_new.db
D:\new_data\resultsNew
```

پوشه `resultsNew` باید خروجی تحلیل همان دیتابیس باشد و فایل‌هایی مثل این‌ها را داشته باشد:

```text
resultsNew\
├── tables\
│   ├── authors.csv
│   ├── papers.csv
│   ├── paper_authors.csv
│   ├── communities.csv
│   ├── collaborations.csv
│   ├── institutions.csv
│   └── countries.csv
├── networks\
│   └── coauthorship.gexf
└── clusters\
    ├── community_fields.csv
    └── community_papers.csv
```

ابتدا index قبلی سامانه را حذف یا جابه‌جا کنید:

```powershell
Remove-Item ".\data\explorer.db" `
  -Force `
  -ErrorAction SilentlyContinue

Remove-Item ".\data\semantic" `
  -Recurse `
  -Force `
  -ErrorAction SilentlyContinue
```

سپس index جدید را بسازید:

```powershell
.\build_explorer_index.bat `
  "D:\new_data\articles_new.db" `
  "D:\new_data\resultsNew"
```

بعد سامانه را اجرا کنید:

```powershell
.\run_development.bat
```

آدرس سامانه:

```text
http://localhost:5173
```

## حالت دوم: فقط دیتابیس جدید را دارید

در این حالت ابتدا باید تحلیل نویسندگان و کلاسترها را برای دیتابیس جدید اجرا کنید.

### مرحله ۱: تولید خروجی تحلیل شبکه

در پروژه `author_bibliometric_analyzer` اجرا کنید:

```powershell
.venv\Scripts\python.exe app.py `
  "D:\new_data\articles_new.db" `
  --output "D:\new_data\resultsNew" `
  --min-author-papers 2 `
  --min-edge-weight 2 `
  --static-network-top-nodes 350 `
  --interactive-network-top-nodes 3000
```

این مرحله فایل‌های زیر را تولید می‌کند:

* پروفایل نویسندگان
* روابط همکاری
* کلاسترها
* مؤسسات
* کشورها
* فایل GEXF
* گزارش Excel
* شبکه‌های تعاملی

### مرحله ۲: استخراج حوزه و مقالات هر کلاستر

```powershell
.venv\Scripts\python.exe -X utf8 extract_cluster_papers.py `
  "D:\new_data\resultsNew"
```

پس از این مرحله فایل‌هایی مانند زیر ساخته می‌شوند:

```text
resultsNew\clusters\community_fields.csv
resultsNew\clusters\community_papers.csv
resultsNew\clusters\community_representative_papers.csv
```

### مرحله ۳: ساخت index سامانه وب

به پوشه پروژه وب بروید:

```powershell
cd D:\research_intelligence_explorer_full\research_intelligence_explorer_full
```

index قبلی را پاک کنید:

```powershell
Remove-Item ".\data\explorer.db" `
  -Force `
  -ErrorAction SilentlyContinue

Remove-Item ".\data\semantic" `
  -Recurse `
  -Force `
  -ErrorAction SilentlyContinue
```

سپس:

```powershell
.\build_explorer_index.bat `
  "D:\new_data\articles_new.db" `
  "D:\new_data\resultsNew"
```

و در نهایت:

```powershell
.\run_development.bat
```

---

## نکته بسیار مهم

پوشه نتایج باید دقیقاً متعلق به همان دیتابیس باشد.

این ترکیب اشتباه است:

```text
articles_new.db
resultsLight مربوط به دیتابیس قبلی
```

چون شناسه مقالات، نویسندگان، کلاسترها و عضویت مقاله‌ها با یکدیگر هماهنگ نخواهند بود.

ترکیب درست:

```text
articles_new.db
resultsNew تولیدشده از articles_new.db
```

## ساختار مورد انتظار دیتابیس جدید

حداقل باید جدول زیر وجود داشته باشد:

```text
articles
```

و ستون ضروری:

```text
authors_json
```

ستون‌های پیشنهادی:

```text
id
title
abstract
publication_year
journal
doi
citation_count
authors_json
```

ستون‌های اختیاری ولی مفید:

```text
keywords
publisher
type
language
volume
issue
page
url
references_count
```

برنامه تحلیل اولیه نام‌های رایج ستون‌ها را خودکار تشخیص می‌دهد. مثلاً برای استناد، نام‌های زیر قابل تشخیص‌اند:

```text
citation_count
citations
is_referenced_by_count
cited_by_count
times_cited
```

اگر نام ستون‌های دیتابیس متفاوت باشد، هنگام اجرای تحلیل اولیه آن‌ها را صریح تعیین کنید:

```powershell
.venv\Scripts\python.exe app.py `
  "D:\new_data\articles_new.db" `
  --output "D:\new_data\resultsNew" `
  --id-column article_id `
  --year-column year `
  --title-column article_title `
  --abstract-column summary `
  --journal-column source_title `
  --doi-column article_doi `
  --citations-column cited_by
```

## تعویض سریع میان چند دیتابیس

برای هر دیتابیس بهتر است یک پوشه نتایج جدا داشته باشید:

```text
D:\research_data\
├── digital_twin\
│   ├── articles.db
│   └── results
├── cyber_security\
│   ├── articles.db
│   └── results
└── 6g\
    ├── articles.db
    └── results
```

اما نسخه فعلی سامانه در هر لحظه یک index فعال دارد:

```text
data\explorer.db
```

برای تعویض حوزه، index را دوباره با دیتابیس موردنظر بسازید.

مثلاً برای حوزه امنیت سایبری:

```powershell
.\build_explorer_index.bat `
  "D:\research_data\cyber_security\articles.db" `
  "D:\research_data\cyber_security\results"
```

برای بازگشت به Digital Twin:

```powershell
.\build_explorer_index.bat `
  "D:\research_data\digital_twin\articles.db" `
  "D:\research_data\digital_twin\results"
```

## پیشنهاد بهتر برای چند دیتابیس

اگر قرار است مرتباً میان چند حوزه جابه‌جا شوید، بهتر است برای هر حوزه index جدا ذخیره شود:

```text
data\
├── digital_twin\
│   ├── explorer.db
│   └── semantic\
├── cyber_security\
│   ├── explorer.db
│   └── semantic\
└── 6g\
    ├── explorer.db
    └── semantic\
```

سپس در رابط کاربری یک انتخاب‌گر دیتاست اضافه شود:

```text
Digital Twin
Cybersecurity
6G
Industrial Control
```

در نسخه فعلی، روش مطمئن برای دیتابیس جدید این زنجیره است:

```text
DB جدید
↓
اجرای author_bibliometric_analyzer
↓
اجرای extract_cluster_papers.py
↓
ساخت explorer index
↓
اجرای سامانه وب
```
