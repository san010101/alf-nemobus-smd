# Оформление шелкографии печатной платы (PCB Silkscreen Artwork)

Для придания плате **ALF-NEMOBUS** уникального ретро-стиля используется декоративная рамка на слое шелкографии (`F.SilkS` / `B.SilkS`), объединяющая космическую тематику планеты Мельмак (сериал *ALF*) и морскую эстетику «Наутилуса» капитана Немо (отсылка к шине **NemoBus**).

---

## Промт для генерации графики (Midjourney / DALL-E 3 / Stable Diffusion)

### English Prompt (Recommended):
> Black and white vector line art for a PCB silkscreen design. A decorative border blending marine and space themes. The top features Melmacian planets, stars, and a Nautilus ship's helm. The bottom and sides include oceanic elements like waves, octopus tentacles, mechanical gears, and technical PCB connectors. In the center, bold retro text reading "ALF-NEMOBUS", alongside the phrases "ZX SPECTRUM ADAPTER BOARD" and "from san010101". High contrast, stencil style, clean monochrome vectors, no gradients.

### Русский промт (Шедеврум / Kandinsky):
> Черно-белая векторная графика для шелкографии печатной платы. Декоративная рамка, объединяющая морскую и космическую тематику. Наверху — планеты Мельмака, звезды и штурвал «Наутилуса». Снизу и по бокам — морские волны, щупальца осьминога, механические шестеренки и технические разъемы. В центре крупный ретро-текст: "ALF-NEMOBUS", а также надписи "ZX SPECTRUM ADAPTER BOARD" и "from san010101". Высокий контраст, стиль трафарета, четкие черные линии на белом фоне, без градиентов.

---

## Рекомендации по импорту в KiCad 10

1. Сгенерируйте изображение и откройте его во встроенном конвертере **KiCad Image Converter** (`Инструменты -> Конвертер изображений`).
2. Выберите формат вывода **Посадочное место (`.kicad_mod`)** на слой **`F.SilkScreen`** (или `B.SilkScreen`).
3. Установите порог черного/белого около `50–60%`, при необходимости включите инверсию, чтобы линии рисунка отображались белым цветом шелкографии на маске платы.
4. Задайте физический размер рамки в миллиметрах под габариты платы `alf-nemobus` и сохраните футпринт в пользовательскую библиотеку `my_lib`.
