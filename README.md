# Lab06 — Category-Partition ба PICT

**Хичээл:** F.CSA313 — Программ хангамжийн чанарын баталгаа ба тест (2026)

**Оюутан:** С. Энхбаяр — **B232270153**

## Орчин ба PICT бүтээх

PICT-ийг WSL2 Ubuntu дээр `make -j4`-өөр бүтээсэн. Эх код `~/tools/pict`, лабораторийн репо `~/projects/F.CSA313_lab6` байрлалд байна.

```bash
mkdir -p ~/tools
cd ~/tools
git clone https://github.com/microsoft/pict.git
cd pict
make -j4
mkdir -p ~/.local/bin
cp pict ~/.local/bin/pict
export PATH="$HOME/.local/bin:$PATH"
pict
```

`pict` командыг аргументгүй ажиллуулахад хэрэглээний заавар хэвлэгдэнэ. Шинэ терминалд PATH-ийн export командыг дахин ажиллуулна.

Source commit, build, хэрэглээний заавар, 2 × 2 туршилтын гаралт: [pict-build.txt](results/pict-build.txt). Туршилтын бүх хос үүссэн, exit 0.

## Category-Partition дизайн

Тестлэх функц: `POST /transfer {"from": ID, "to": ID, "amount": NUM, "currency": CODE}`. Кодгүй тул oracle-ууд нь хүлээгдэх үр дүн юм.

### Төлөөлөх орчин

Эх данс `ACC_FROM`, хүлээн авагч `ACC_TO`, байхгүй данс `ACC_MISSING` гэж авна. Байгаа дансны валют MNT, эх дансны үлдэгдэл 2,000 MNT, өдрийн нийт лимит 3,000 MNT байна. `Frozen` үед эх данс царцаагдсан; `SameAsFrom` үед `to=from` (байгаа эх дансны хувьд `ACC_FROM`).

| Далд орчны сонголт | Төлөөлөх утга | Холбох нөхцөл |
|---|---|---|
| Өдрийн лимитийн ашиглалт | `Unused`: ашигласан 0, үлдсэн 3,000 MNT | `[IF] Amount != OverLimit` |
| Өдрийн лимитийн ашиглалт | `NearLimit`: ашигласан 2,500, үлдсэн 500 MNT | `[IF] Amount = OverLimit` |
| Үлдэгдэл | 2,000 MNT | `[property]` Бүх кейсийн тогтмол setup |
| Хөрвүүлэх ханш | 1 USD = 2 MNT | `[IF] Currency = Different`; тестийн тогтмол ханш |

`amount` нь хүсэлтийн валютаар; үлдэгдэл, лимитийг MNT-д хөрвүүлж шалгана. USD дүн нь MNT төлөөлөх дүнгийн хагас; ханш нь тестийн тогтмол утга юм.

### Сонголт, утга ба хязгаарлалт

| Сонголт | Утга | Төлөөлөх утга | Тэмдэглэгээ ба үндэслэл |
|---|---|---|---|
| From | `Valid` | `ACC_FROM`, царцаагдаагүй | `[property FromExists, Usable]` Хүчинтэй эх данс |
| From | `Frozen` | `ACC_FROM`, царцаагдсан | `[ERROR]` Шилжүүлэг хийх эрхгүй |
| From | `Missing` | `ACC_MISSING`, үүсгээгүй | `[ERROR]` Эх данс байхгүй |
| To | `Valid` | `ACC_TO`, эх данснаас өөр | `[property ToExists, Distinct]` Хүчинтэй хүлээн авагч |
| To | `Missing` | `ACC_MISSING`, үүсгээгүй | `[ERROR]` Хүлээн авагч байхгүй |
| To | `SameAsFrom` | `ACC_FROM` | `[ERROR, IF FromExists]` Өөрийн данс руу шилжүүлэхийг хориглоно |
| Amount | `Zero` | 0 MNT | `[ERROR]` amount > 0 нөхцөл зөрчинө |
| Amount | `Normal` | Хөрвүүлсний дараа 100 MNT | `[property Positive, WithinBalance, WithinLimit]` Амжилтын үндсэн дүн |
| Amount | `AtBalance` | 2,000 MNT | `[SINGLE, IF Currency=Same]` Үлдэгдэлтэй яг тэнцэх хязгаарыг нэг удаа шалгана |
| Amount | `OverBalance` | 2,500 MNT | `[ERROR, IF Unused]` Үлдэгдлээс хэтэрнэ, өдрийн лимитээс хэтрэхгүй |
| Amount | `OverLimit` | 1,000 MNT | `[ERROR, IF NearLimit]` Үлдэгдэл хүрэлцэнэ, үлдсэн өдрийн лимитээс хэтэрнэ |
| Currency | `Same` | MNT | `[property NoConversion]` Хөрвүүлэлт шаардлагагүй |
| Currency | `Different` | USD, 1 USD = 2 MNT | `[IF FromExists]` Хөрвүүлсэн дүнгээр шалгана |

ERROR кейс бүр нэг алдааны утгатай; бусдыг `Valid / Valid / Normal / Same` суурь утгад тогтооно. Amount алдаатай үед Normal-ийг тухайн утгаар солино.

SINGLE нь `Valid / Valid / AtBalance / Same` нэг кейс. Давуу эрэмбэд зориулж `Frozen / Missing / Normal / Same` давхар нөхцөлийг нэмсэн. Лимитийн setup нь Amount-тай холбогдох тул тусдаа үржүүлэгч биш.

### Олонлогийн тооцоо

Хязгаарлалтгүй дөрвөн сонголтын үржвэр: **3 × 3 × 5 × 2 = 90**.

| Хязгаарлалтын дараах бүлэг | Тоо |
|---|---:|
| `Valid / Valid / Normal`, хоёр валют | 2 |
| Нэг ERROR утгатай, бусад нь үндсэн утга | 7 |
| `AtBalance` SINGLE | 1 |
| Үндсэн Category-Partition олонлог | **10** |
| Зориуд нэмсэн давхар алдаа | 1 |
| Эцсийн спецификаци | **11** |

**90 → 10**, давхар нөхцөл нэмээд **11 гарын спецификаци** болно. Энэ нь PICT-ийн мөрийн тоо биш; валютын ялгааг үндсэн амжилтын кейсүүдэд хослуулсан.

### Алдааны давуу эрэмбэ

`ERROR_NO_FROM → ERROR_FROZEN → ERROR_NO_TO → ERROR_SAME_ACCOUNT → ERROR_BAD_AMOUNT → ERROR_INSUFFICIENT → ERROR_LIMIT`.

Дансны оршин байдал, эрхийг мөнгөн дүнгээс өмнө шалгахаар сонгосон. Энэ нь дизайны шийдвэр, туршилтаар тогтоосон эрэмбэ биш.

### Спецификацийн хүснэгт

`amount` нь хүсэлтийн дүн, ашигласан лимит нь MNT-ээр. Amount классыг хөрвүүлсний дараа үнэлнэ.

| Кейс | From | To | Amount | Currency | amount | Ашигласан лимит | Статус | result |
|---|---|---|---|---|---:|---:|---:|---|
| CP01 | Valid | Valid | Normal | Same (MNT) | 100 | 0 | 201 | `OK` |
| CP02 | Valid | Valid | Normal | Different (USD) | 50 | 0 | 201 | `OK` |
| CP03 | Frozen | Valid | Normal | Same (MNT) | 100 | 0 | 200 | `ERROR_FROZEN` |
| CP04 | Missing | Valid | Normal | Same (MNT) | 100 | 0 | 200 | `ERROR_NO_FROM` |
| CP05 | Valid | Missing | Normal | Same (MNT) | 100 | 0 | 200 | `ERROR_NO_TO` |
| CP06 | Valid | SameAsFrom | Normal | Same (MNT) | 100 | 0 | 200 | `ERROR_SAME_ACCOUNT` |
| CP07 | Valid | Valid | Zero | Same (MNT) | 0 | 0 | 200 | `ERROR_BAD_AMOUNT` |
| CP08 | Valid | Valid | OverBalance | Same (MNT) | 2500 | 0 | 200 | `ERROR_INSUFFICIENT` |
| CP09 | Valid | Valid | OverLimit | Same (MNT) | 1000 | 2500 | 200 | `ERROR_LIMIT` |
| CP10 | Valid | Valid | AtBalance | Same (MNT) | 2000 | 0 | 201 | `OK` |
| CP11 | Frozen | Missing | Normal | Same (MNT) | 100 | 0 | 200 | `ERROR_FROZEN` |

CP02: 50 USD → 100 MNT; CP08: үлдэгдэл дутна, лимит хүрэлцэнэ; CP09: үлдэгдэл хүрэлцэнэ, лимит дутна. CP10 нь SINGLE; CP11-д `ERROR_FROZEN` түрүүлнэ.

Байхгүй эх дансны frozen төлөв, байхгүй хүлээн авагчийн SameAsFrom шинж нь утгагүй хослол тул үндсэн олонлогт орохгүй.

## Браузерын PICT модель

`browser.model` нь `Content`, `Cookies` тус бүр Allow/Restrict/Block, `Popups`, `Addons`, `Attacks`, `Forgeries` тус бүр Yes/No гэсэн утгатай. Бүрэн олонлог **3 × 3 × 2 × 2 × 2 × 2 = 144**.

```bash
export PATH="$HOME/.local/bin:$PATH"
pict browser.model > results/browser-2way.txt
pict browser.model /o:3 > results/browser-3way.txt
tail -n +2 results/browser-2way.txt | wc -l
tail -n +2 results/browser-3way.txt | wc -l
```

| Хамралт | Өгөгдлийн мөр | Хамрагдсан хослол | Бүтэн гаралт |
|---|---:|---|---|
| 2-way | 10 | Бүх 81 хос | [browser-2way.txt](results/browser-2way.txt) |
| 3-way | 20 | Бүх 248 гурвал | [browser-3way.txt](results/browser-3way.txt) |
| Бүрэн олонлог | 144 | Бүх зургаан параметрийн хослол | Онолын үржвэр |

Толгой мөрийг тоолоогүй; хамралтыг шалгасан, exit 0. `/r` ашиглаагүй, давтан гаралтууд ижил.

PICT 2-way нь лекцийн 9 мөрөөс нэгээр олон буюу 10 мөр өгсөн. Greedy арга хамгийн цөөн мөрийг заавал олдоггүй. 3-way-ийн 20 мөр нь бүрэн олонлогийн 144-өөс цөөн.

## Шилжүүлгийн PICT моделиуд

`transfer-nc.model` нь хязгаарлалтгүй; `transfer.model` нь ERROR/SINGLE болон давхар нөхцөлийн IF дүрэмтэй. `transfer-neg.model` нь ижил дүрэмтэй, долоон ERROR утгыг `~`-ээр тэмдэглэсэн.

```bash
export PATH="$HOME/.local/bin:$PATH"
pict transfer-nc.model > results/transfer-nc.txt
pict transfer.model > results/transfer.txt
pict transfer-neg.model > results/transfer-neg.txt
```

| Модель | Өгөгдлийн мөр | Happy path | Нэг мөр дэх ERROR утга | Олон алдаатай мөр | Зөвшөөрөгдсөн хосын хамралт | Гаралт |
|---|---:|---:|---|---:|---|---|
| Хязгаарлалтгүй | 15 | 0 | 1–3 | 10 | 61 / 61 | [transfer-nc.txt](results/transfer-nc.txt) |
| IF хязгаарлалттай | 10 | 2 | 0–2 | 1 | 34 / 34 | [transfer.txt](results/transfer.txt) |
| IF + negative утгатай | 10 | 3 | 0–1 | 0 | 33 / 33 | [transfer-neg.txt](results/transfer-neg.txt) |

Толгой мөрийг тоолоогүй; exit 0, давтан гаралт ижил. ERROR тоо нь From-ийн Frozen/Missing, To-ийн Missing/SameAsFrom, Amount-ийн Zero/OverBalance/OverLimit утгуудын тоо; эдгээргүй мөрийг happy path гэж тоолсон.

Хязгаарлалтгүй гаралт happy path үүсгээгүй; олон алдаатай мөрөнд эхний алдаа бусдыг нууна (error masking). IF модельд Frozen + Missing давхар нөхцөл нэг мөрөөр орсон.

IF домэйнд 11 хослол байгаа ч PICT 10 мөрөөр бүх зөвшөөрөгдсөн хосыг хамарсан; CP01 Normal/Same мөр тусдаа гараагүй. Negative модель хоёр алдаатай хослолыг хасаж, 10 хослолыг 10 мөрөөр гаргасан. `~` нь SINGLE, Currency, setup-ийг тогтоохгүй тул IF дүрэм хэрэгтэй хэвээр.

IF нь сонгосон тестийн бодлого тул 34/33 хосын хамралтыг зөвшөөрөгдсөн домэйнд үнэлсэн. Хасагдсан хослол бүр бодит системд боломжгүй гэсэн үг биш; лимит, ханшийг дээрх setup-аар тогтооно.

### Гаралтаас бодит кейс боловсруулах

Мөрийн дугаарт толгой мөр ороогүй; `~`-ийг API оролтод илгээхгүй. Доорх нь setup, оролт, хүлээгдэх oracle бүхий дизайн, бодит POST үр дүн биш.

| Кейс ба эх мөр | Setup | POST /transfer body | Oracle |
|---|---|---|---|
| P01, transfer.txt мөр 2: Valid / Valid / Normal / Different | ACC_FROM идэвхтэй, 2,000 MNT; ACC_TO байгаа; өдрийн лимит 3,000, ашигласан 0; ханш 1 USD = 2 MNT | `{"from":"ACC_FROM","to":"ACC_TO","amount":50,"currency":"USD"}` | Статус 201; `result="OK"` |
| P02, transfer-neg.txt мөр 8: Valid / Valid / ~OverLimit / Same | ACC_FROM идэвхтэй, 2,000 MNT; ACC_TO байгаа; өдрийн лимит 3,000, ашигласан 2,500 буюу үлдсэн 500 | `{"from":"ACC_FROM","to":"ACC_TO","amount":1000,"currency":"MNT"}` | Статус 200; `result="ERROR_LIMIT"` |
| P03, transfer.txt мөр 5: Frozen / Missing / Normal / Same | ACC_FROM царцаагдсан, 2,000 MNT; ACC_MISSING үүсгээгүй; өдрийн лимит 3,000, ашигласан 0 | `{"from":"ACC_FROM","to":"ACC_MISSING","amount":100,"currency":"MNT"}` | Статус 200; `result="ERROR_FROZEN"` |

P01: 50 USD → 100 MNT. P02: 1,000 > 500 тул лимит дутна. P03: сонгосон эрэмбээр ERROR_FROZEN түрүүлнэ.

## Дүгнэлт

Category-Partition нь алдааны семантик, хязгаарын кейсийг тодорхойлдог.
ERROR/SINGLE/IF нөхцөлийг уялдуулах нь хамгийн их бодол шаардсан.
Валютын хөрвүүлэлт, ашигласан лимитийг setup-д тусгах хэрэгтэй.
Гарын хүснэгт алдааны анги, happy path, давхар нөхцөлийг ойлгомжтой харуулсан.
PICT нь зөвшөөрөгдсөн хосын хамралтыг автоматаар хангадаг.
Pairwise хамралт happy path-ийг заавал үүсгэдэггүйг ажигласан.
Олон алдаатай мөрөнд эхний алдаа бусдыг нуудаг.
Negative утга давхцлыг арилгах ч SINGLE болон IF-ийг орлохгүй.
Greedy арга хамгийн бага мөрийг батлахгүй тул хамралтыг шалгасан.
API кодгүй учраас дизайны нийцлийг үнэлсэн, бодит системийн согог илэрсэн гэж дүгнээгүй.
