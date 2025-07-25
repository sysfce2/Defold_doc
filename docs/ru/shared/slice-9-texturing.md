## Текстурирование Slice9

В графических интерфейсах часто встречаются элементы, чувствительные к контексту в отношении их размера: панели и диалоговые окна, размер которых необходимо изменять, чтобы вместить содержащееся в них содержимое. Это может вызвать визуальные проблемы, если применять текстурирование к изменяемой в размерах ноде.

Обычно движок масштабирует текстуру, чтобы она соответствовала границам ноды Box, но, определив краевые области Slice9, можно определить границы того, какие части текстуры должны масштабироваться:

![GUI scaling](../shared/images/gui_slice9_scaling.png)

Нода Box *Slice9* включает в себя 4 числа, которые определяют количество пикселей для левого, верхнего, правого и нижнего полей, которые не должны подвергаться регулярному масштабированию:

![Slice 9 properties](../shared/images/gui_slice9_properties.png)

Поля устанавливаются по часовой стрелке, начиная с левого края:

![Slice 9 sections](../shared/images/gui_slice9.png)

- Угловые сегменты никогда не масштабируются.
- Краевые сегменты масштабируются вдоль одной оси. Левый и правый краевые сегменты масштабируются по вертикали. Верхний и нижний краевые сегменты масштабируются по горизонтали.
- Центральная область текстуры масштабируется по горизонтали и вертикали по мере необходимости.

Описанное выше масштабирование текстуры *Slice9* применяется только при изменении размера ноды Box:

![GUI box node size](../shared/images/gui_slice9_size.png)

![Sprite size](../shared/images/sprite_slice9_size.png)

::: important
Если вы изменяете параметр масштабирования (scale) у ноды Box или спрайта (или у игрового объекта) — нода, спрайт и текстура масштабируются без применения параметров *Slice9*.
:::

::: important
При использовании Slice9-текстурирования со спрайтами параметр [Sprite Trim Mode изображения](https://defold.com/manuals/atlas/#image-properties) должен быть установлен в значение Off.
:::

### Mipmaps и Slice9

Из-за особенностей работы mipmapping в рендерере масштабирование сегментов текстуры может вызывать артефакты. Это происходит при _уменьшении масштаба_ сегментов ниже исходного размера текстуры. В этом случае рендерер выбирает mipmap более низкого разрешения для сегмента, что приводит к визуальным артефактам.

![Slice 9 mipmapping](../shared/images/gui_slice9_mipmap.png)

Чтобы избежать этой проблемы, убедитесь, что сегменты текстуры, которые будут масштабироваться, достаточно малы, чтобы их не приходилось уменьшать, а только увеличивать.
