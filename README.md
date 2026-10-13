# Lab06 — Category-Partition ба PICT

**Хичээл:** F.CSA313 — Программ хангамжийн чанарын баталгаа ба тест (2026)

**Оюутан:** С. Энхбаяр — **B232270153**

## Орчин ба PICT бүтээх

Microsoft PICT-ийг WSL2 Ubuntu дээр эх кодоос бүтээсэн. CMake суулгаагүй тул бэлэн g++, Make ашиглан зааврын Make хувилбараар бүтээв. PICT-ийн эх код `~/tools/pict`, лабораторийн репо `~/projects/F.CSA313_lab6` байрлалд байна.

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

Source commit, compiler болон Make хувилбар, build гаралт, хэрэглээний заавар, хоёр параметртэй туршилтын гаралтыг [pict-build.txt](results/pict-build.txt)-д хадгалсан. Туршилтын загварын 2 × 2 боломжит хосыг PICT бүрэн үүсгэж, exit code 0 буцаасныг шалгасан.

## Category-Partition дизайн

Тестлэх функц: `POST /transfer {"from": ID, "to": ID, "amount": NUM, "currency": CODE}`. Код байхгүй тул доорх хүснэгтүүд нь тодорхойлолтоос гаргасан дизайн, хүлээгдэх үр дүн юм.

### Төлөөлөх орчин

Эх данс `ACC_FROM`, хүлээн авагч `ACC_TO`, байхгүй данс `ACC_MISSING` гэж авна. Байгаа дансны валют MNT, эх дансны үлдэгдэл 2,000 MNT, өдрийн нийт лимит 3,000 MNT байна. `Frozen` үед эх данс царцаагдсан; `SameAsFrom` үед `to=ACC_FROM`.

| Далд орчны сонголт | Төлөөлөх утга | Холбох нөхцөл |
|---|---|---|
| Өдрийн лимитийн ашиглалт | `Unused`: ашигласан 0, үлдсэн 3,000 MNT | `[IF] Amount != OverLimit` |
| Өдрийн лимитийн ашиглалт | `NearLimit`: ашигласан 2,500, үлдсэн 500 MNT | `[IF] Amount = OverLimit` |
| Үлдэгдэл | 2,000 MNT | `[property]` Бүх кейсийн тогтмол setup |
| Хөрвүүлэх ханш | 1 USD = 2 MNT | `[IF] Currency = Different`; тестийн тогтмол ханш |

`amount` нь хүсэлтийн `currency`-гийн валютаар байна. Үлдэгдэл, лимитийг MNT рүү хөрвүүлсэн дүнгээр шалгана; USD хүсэлтийн дүн нь хүснэгтийн MNT төлөөлөх дүнгийн хагас байна. Ханш нь бодит зах зээлийн ханш биш.

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

Ердийн ERROR кейс бүрд зөвхөн нэг алдааны утга сонгож, бусад сонголтыг үндсэн хүчинтэй утгад тогтооно: `From=Valid`, `To=Valid`, `Amount=Normal`, `Currency=Same`. Amount-ийн ERROR өөрөө сонгогдсон үед Normal-ийг тухайн утгаар солино. ERROR кейсийн Normal нь бусад мөнгөн дүнгийн алдааг тусгаарлах суурь утга болно.

SINGLE кейс нь `Valid / Valid / AtBalance / Same`; бусад сонголтоор дахин үржүүлэхгүй. Хоёр алдаатай `Frozen / Missing / Normal / Same` хослолыг зөвхөн давуу эрэмбийн тусгай спецификацид зориуд зөвшөөрнө. Өдрийн лимитийн ашиглалт нь Amount-тай холбогдсон setup тул тусдаа чөлөөт үржүүлэгч гэж тооцохгүй.

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

Ингэснээр **90 → 10**, давуу эрэмбийн нэг тусгай кейс нэмэхэд **11** болно. Энэ нь гарын дизайны сонгосон олонлогийн тоо; PICT-ийн pairwise гаралтын мөрийн тоо биш. Хязгаарлалтууд алдааг тусгаарлаж, валютын ялгааг зөвхөн үндсэн амжилтын кейсүүдэд хослуулна.

### Алдааны давуу эрэмбэ

`ERROR_NO_FROM → ERROR_FROZEN → ERROR_NO_TO → ERROR_SAME_ACCOUNT → ERROR_BAD_AMOUNT → ERROR_INSUFFICIENT → ERROR_LIMIT`.

Эхлээд дансны оршин байдал ба эрхийг шалгаж, дараа нь мөнгөн дүнг үнэлэхээр сонгосон нь байхгүй эсвэл царцаагдсан дансны дараагийн нөхцөлийг үнэлэхээс сэргийлнэ. Энэ нь дизайнд сонгосон эрэмбэ бөгөөд кодгүй тул туршилтаар тогтоосон гэж үзэхгүй.

### Спецификацийн хүснэгт

`amount` багана нь хүсэлтийн бодит дүн, `Ашигласан лимит` нь MNT-ээрх setup утга юм. Хүснэгтийн Amount классыг хөрвүүлсний дараах дүнгээр тодорхойлсон.

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

CP02-д 50 USD → 100 MNT хөрвүүлэгдэж, үлдэгдэл болон лимит хүрэлцэнэ. CP08-д 2,500 > 2,000 боловч 2,500 ≤ 3,000 тул зөвхөн үлдэгдлийн алдаа гарна. CP09-д 1,000 ≤ 2,000 боловч 1,000 > 500 тул зөвхөн лимитийн алдаа гарна. CP10 нь SINGLE хязгаарын кейс; CP11-д Frozen ба Missing давхацсан ч сонгосон эрэмбээр `ERROR_FROZEN` хүлээнэ.

Байхгүй эх дансны frozen төлөв, байхгүй хүлээн авагчийн SameAsFrom шинжийг үнэлэх нь утгагүй хослол юм. Ийм нөхцөлийг үндсэн сонгосон олонлогт оруулахгүй; зориудын давхар нөхцөлийг тусад нь тэмдэглэсэн.

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

Мөрийн тоонд параметрийн нэртэй эхний мөр ороогүй. Хоёр ажиллуулалтын exit code 0; хамралтыг бүх шаардлагатай хос, гурвалын олонлогтой харьцуулж баталгаажуулсан. `/r` ашиглаагүй бөгөөд хоёр гаралт давтан ажиллуулахад byte-for-byte ижил байсан.

Лекцийн гараар байгуулсан 9 мөртэй харьцуулахад PICT 2-way нь 10 мөр буюу нэгээр олон өгсөн. PICT-ийн greedy арга хамралтыг хангах боловч хамгийн цөөн мөртэй шийдлийг заавал олдоггүй. 3-way нь илүү өндөр хамралтад 20 мөр шаардаж байгаа ч бүрэн олонлогийн 144 мөрөөс цөөн байна.
