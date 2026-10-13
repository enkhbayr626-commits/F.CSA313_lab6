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
