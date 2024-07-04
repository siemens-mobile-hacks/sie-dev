# Changelog
## 2024.07.01 - [@Azq2](https://github.com/Azq2)

1. В состав ElfPack включены дополнительные патчи:
      - Исправлен перезапуск IDLE CSM при ошибке SIM или смене IMSI. Это актуально для multi-IMSI SIM, в которых IMSI может меняться во время работы.
      - Запатчены все функции BSD-сокетов, чтобы исключить возможное состояние гонки (если процесс NETCORE тормозит), что приводило к повреждению стека.
2. Теперь все модели и версии прошивки из линейки NSG/ELKA получили:
      - Правильную область RAM, которая точно не используется прошивкой.
      - Врезка в инициализацию ОС для зануления этой несипользуемой памяти.
      - Перенос зеркала swilib и конфига из HEAP в неиспользуемую память (экономия 18 кб хипа).
3. Инструменты для автоматического генерирования всех патчей, что перечислены выше.

## 2024.03.14 - [@Azq2](https://github.com/Azq2)
- Изменена область RAM используемая ELFLoader для S75sw52, C81sw51, EL71sw45, E71sw45.

  Теперь используется точно неиспользуемая ранее память.

  Так же добавлена врезка в инициализацию ОС для зануления этого блока памяти.

  Спасибо Feyman за предоставленные адреса несипользуемой памяти и FIL, k1r1t0 за тесты.
- Для S75sw52, C81sw51, EL71sw45, E71sw45 зеркало swilib и конфиг перенесены в статическую память, вместо HEAP.
  Это позволило сэкономить до 18 кб heap-памяти.
- Добавлена защита от двойного запуска ELFLoader и удалён его деструктор (в нём нет смысла).
- Исправлена работа Explorer, которая была сломана из-за убранного `__thumb` у функции `MyShowMSG`.
## 2024.02.28 - [@Azq2](https://github.com/Azq2)
- Исправлена область RAM используемая ELFLoader для S75sw52
- Исправлена функция FUNC_ABORT, у которой случайно было убратно `__arm`, из-за этого вместо "No function in lib" был пик.

## 2024.02.20 - [@Azq2](https://github.com/Azq2)

**Багфиксы:**
1. Заменил все "way" на "path"
2. Исправил пик при клике на .elf, который не является эльфом (часто бывает, что файл нулевого размера и при клике на него пикало).
3. Исправлен баг swi.blib, когда она не использовалась для EP3 эльфов, всегда передавался указатель на library.vkp.
4. Исправлен баг, когда в `R_ARM_ABS32` не учитывался предварительный offset.
5. Добавлены данные для восстановления, чтобы можно было нормально откатывать патч ElfPack.
6. Удалена AddrLibrary из патча ELFLoader.

**Изменения:**
1. Теперь "Realtime lib cache cleaner" включен по-умолчанию.
2. Добавлено автоматическое изменение номера диска в конфиге Elfloader (при первом запуске).
  
   Раньше могла быть ситуация, когда конфиг создаётся на `0:\Zbin\etc\Elfpack3.bcfg`, но пути в нём остаются `4:\...`.
  
   Теперь конфиг создаётся с правильным диском.
3. `__sys_switab_addres` и `__switab` теперь предоставляются самим ELFLoader, вместо libcrt_helper.so.
4. Встроенные символы `__ex`, `__sys_switab_addres` и `__switab` теперь работают так же и в `R_ARM_GLOB_DAT`.
5. Копирование библиотеки функций в RAM перенесено из libcrt_helper.so в сам ELFLoader.
6. Все несуществующие функции теперь не 0xFFFFFFFF, а 0xFFFFxxx0, где **xxx** это swi-номер.
   
   Если раньше при попытке доступа к неизвестной функции происходил непонятный дата-аборт (в GCC-эльфах), то теперь будет явная ошибка доступа к адресу 0xFFFFxxx0, из которого можно понять, какая именно из функций отсутствует.
   
   При этом проблем не будет, в swilib никогда не указывались адреса или данные 0xFFFFxxxx.
7. Базовая поддержка для реализации gdbstub.
   - Возможность установить хук на загрузку elf/so.
   - Первым эльфом грузится 0:\Zbin\Daemons\EP3DebugHook.elf (если он присутствует на диске), в него передаётся указатель на ElfloaderDebugHook, в который можно будет прописать указатели на `hook()` и `_r_debug`.

8. Удалось сократить размер EP3 почти на 0.5-0.7 кб.
9. Уменьшено кол-во malloc/free при загрузке ELF.
10. Удалены бесполезные функции из swilib: elfclose, elfopen, elf_entry, GetBinSize, LoadSections, DoRelocation.
11. PT_DYNAMIC для GCC-эльфов теперь работает по стандарту, а для IAR'овских оставлен костыль с клонированием в раму (это нужно для gdbstub).
12. Новая система сборки.
13. Опциональный пропуск загрузки демона, если присутствует файл с его именем с постфиксом .skip (например: `0:\Zbin\Daemons\XTASK3_ELKA.elf.skip`).
    Это нужно, чтобы можно было управлять автозагрузкой без переноса файлов (для будущего пакетного менеджера).

**Изменения работы swi.blib**
1. Теперь можно переопределять почти всю библиотеку функций, а не только добавлять новые функции.
2. При этом добавлена проверка на совместимость. Если некоторые базовые функции (strcat, strchr, strcmp, strcpy) не совпадают с прошивочными, значит, это swi.blib от другого телефона. Он не будет загружен.
3. Safe-Mode (# при загрузке) теперь действует и на swi.blib. Раньше кривая swi.blib могла убить телефон.

## 2023.10.21 - [@Azq2](https://github.com/Azq2)

**Багфиксы:**
- Исправлена сборка EP3 для устаревших прошивок.

## 2023.10.21 - [@Azq2](https://github.com/Azq2)

**Багфиксы:**
- Включен GStackAlign=1 для совместимости с AAPCS (8-байт стек)
- Починена сборка EP3, добавлены отсутствующие файлы

## 2018.12.28 - [@Azq2](https://github.com/Azq2)

**Багфиксы:**
- Исправлен адрес StoreErrString для C81sw51

## 2012.02.02 - [@Alexious-sh](https://github.com/Alexious-sh) [@zvova7890](https://github.com/zvova7890)

- fix dlopen(Data Abort)
- поиск кеш картинок происходит по хешу имени, должно немного дать прироста

## 2012.01.08 - [@Alexious-sh](https://github.com/Alexious-sh) [@zvova7890](https://github.com/zvova7890)

- Добавлен поиск либ в корне папки с эльфом
- Исправлена утечка в логе

## 2011.12.11 - [@Alexious-sh](https://github.com/Alexious-sh) [@zvova7890](https://github.com/zvova7890)

- Исправлена серьёзная ошибка из за которой не правильно импортились некоторые функции из либ, обычно собраных на С++. К примеру не работал fstream из stdc++. Я сперва думал что это глюк в либе, но когда линканул статически то удивился что оно заработало, и стал искать ошибку в паке
- По притензиям некоторых, добавлена в конфиг настройка путей к папкам либ
- Добавлен в конфиг опциональный лог. Так как сложно выловить в мессагах че там не хватает эльфу. Чтобы выключить лог, ставим максимальный размер на 0.

## 2011.??.?? - [@Alexious-sh](https://github.com/Alexious-sh) [@zvova7890](https://github.com/zvova7890)

- Улучшена совместимость со старыми эльфами
- Поддержка симлинков для либ, симлинк обычный текстовый файл содержащий только путь
- Поменян прицыпе вызова конструкторов, они теперь вызываются библиотекой libcrt, разрабам следует обновиться из темы разработки
- Исправлено 2 ошибки в освобождении либ
- Добавлено опциональная очистка либ при не использовании, вы можете выключить реалтайм освобождение тем самым увеличить скорость последующего запуска и освобождения эльфов.
- В свилиб вынесены 4 новых функций
	- 0x2F6 getBaseEnviron - указатель на переменную окружения(требуется для либц)
	- 0x2F7 dlerror        - при ошибке открытия либы эта функция возвратит ошибку в текстовом виде
	- 0x2F8 dlclean_cache  - принудительная очистка кеша либ(если отключена реалтайм очистка)
	- 0x2F9 SHARED_TOP     - указатель на последнюю загруженую либу

## 2011.07.11 - [@Alexious-sh](https://github.com/Alexious-sh) [@zvova7890](https://github.com/zvova7890)

Первая публичная версия
