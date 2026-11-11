# Big Data Engine Benchmark

Pandas, Dask, Spark, Polars и DuckDB на NYC Yellow Taxi.

**Данные:** NYC Yellow Taxi
**Задача:** количество поездок, средний тариф и суммарная дистанция по гео-ячейке и часу суток.
**Сценарий:** подготовка профиля спроса для планирования смен, прогноз к 06:00.
**Вычисления:** локальный CPU; GPU не требуется.

## Запуск

Python 3.13, JDK 17. Исходные CSV размещаются в `data/raw/`,
единая Parquet-копия - в `data/parquet/`.
Источник: [Kaggle / NYC TLC](https://www.kaggle.com/datasets/elemento/nyc-yellow-taxi-trip-data),
январь 2015 и январь–март 2016.

```bash
uv sync
uv run --with jupyterlab jupyter lab
```

Откройте `notebooks/benchmark.ipynb`, выберите `.venv`, выполните Run All.
Упаковка CSV в Parquet, очистка, проверки, замеры и графики выполняются в ноутбуке.

Без uv: `python3.13 -m venv .venv`, затем `.venv/bin/pip install -r requirements.txt`.
Для точного воспроизведения окружения есть `uv.lock`.

### Java

Используйте установленный JDK 17 и `JAVA_HOME`. При подготовке этой работы Temurin 17
установлен только внутри `.venv/java/`, без изменения системной Java.
Ноутбук подхватывает этот локальный JDK, если `JAVA_HOME` не задан.
После пересоздания `.venv` JDK нужно установить снова. Например, на macOS ARM:

```bash
mkdir -p .venv/java
curl -fL 'https://api.adoptium.net/v3/binary/latest/17/ga/mac/aarch64/jdk/hotspot/normal/eclipse' -o .venv/java/jdk.tar.gz
tar -xzf .venv/java/jdk.tar.gz -C .venv/java
```

## Структура

```text
notebooks/benchmark.ipynb  # код, проверки, таблицы и графики
data/                     # локальные очищенные данные, не для Git
```

## Методика

- Одинаковые очищенные данные для всех движков. Срезы 100 МБ / 1 ГБ - логические:
  40 байт на строку (три int64 и два float64), а не объём сжатого Parquet.
- На полном наборе: Spark и контрольная реализация Polars. Pandas защищён лимитом до чтения.
- Пиковый RSS дерева процессов, включая Java; CPU 100 % соответствует одному ядру.
  Дисковые read/write bytes: psutil, на macOS - нативная libproc через ctypes.
  Число операций на macOS недоступно: NaN. Чтение из кеша ОС может дать 0 дисковых байт.
  Это не логический объём данных; объёмы Spark scan/shuffle - из event log.
- Число потоков по умолчанию 4, переопределяется `LAB_CORES`. Кэш ОС не сбрасывается.
- Партиционирование, AQE и repartition тестируются независимо; salting восстанавливает среднее через суммы и количества.
- Все результаты сверяются автоматически. Оценки стоимости инженеров и VM явно отделены от измерений.

Результаты, физические планы, графики и выводы сохранены в ноутбуке по разделам 1–7.
Промежуточные файлы замеров создаются во временном каталоге и удаляются после выполнения.
