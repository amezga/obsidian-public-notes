# Контекстные менеджеры в Python

Контекстные менеджеры — это способ управления ресурсами (файлы, соединения, блокировки и др.) с гарантией освобождения даже при возникновении исключений.

Обычно используются через конструкцию `with`, которая автоматически вызывает методы `__enter__()` при входе и `__exit__()` при выходе из блока.

## Зачем нужны контекстные менеджеры?

- Гарантированное освобождение ресурсов  
- Чистый и читаемый код  
- Безопасность при работе с внешними ресурсами  

Примеры использования:
```python
with open("file.txt") as f:
    data = f.read()
```

---

## Как работает `with`

Конструкция `with` вызывает `__enter__()` при входе и `__exit__()` при выходе из блока. В переменную, которая стоит после **as** записывается значение, возвращаемое из `__enter__`

```python
class DummyContextManager:
    def __enter__(self):
	    print("Enter context manager")
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
	    print("Exit context manager")
        pass

with DummyContextManager() as context:
    print(context)  # 42
```
Хочется обратить внимание на параметры метода `__exit__`
Как упоминалось выше, при выходе из `with` блока всегда вызывается `__exit__`, даже если возникло исключение. Поэтому параметры метода сожержат:
- `exc_type` — тип исключения  
- `exc_val` — объект исключения  
- `exc_tb` — traceback  

---
## Возвращаемое значение

Значение, которое возвращается из `__enter__ ` не всегда является объектом контекстного менеджера
```python
class DummyContextManager:
  def __init__(self):
    pass

  def __enter__(self):
    return 42

  def __exit__(self, exc_type, exc_val, exc_tb):
      pass

with DummyContextManager() as context:
    print("What is the purpose of this context manager?")
    print(context) # >> 42

context_manager = DummyContextManager()  
with context_manager as context:  
    print(context_manager)  # >> <__main__.DummyContextManager object at 0x>
    print(context)          # >> 42
```


---

## Несколько контекстных менеджеров

С Python 3.10 можно писать:

```python
with (
    DummyContextManager() as ctx1,
    DummyContextManager() as ctx2
):
    pass
```

---

## `@contextmanager` из `contextlib`

Декоратор `@contextmanager` превращает генераторную функцию в полноценный контекстный менеджер:

```python
@contextlib.contextmanager
def dummy_context_manager():
    print("Enter")
    yield 42
    print("Exit")

with dummy_context_manager() as ctx:
    print(ctx)
    # >> Enter
    # >> 42
```

⚠️ Исключения внутри `with` не обрабатываются автоматически — `__exit__` будет вызван, но ошибка не будет подавлена.


---

## Стандартная библиотека `contextlib`

Библиотека `contextlib` содержит готовые контекстные менеджеры и инструменты для создания собственных. Подробнее — в [официальной документации](https://docs.python.org/3/library/contextlib.html).

---

## Пример: запись в CSV-файлы с использованием контекстных менеджеров

### Интерфейс записи

```python
class IFile(abc.ABC):
    @abc.abstractmethod
    def write_rows(self, rows: list[dict[str, Any]]) -> None:
        pass

    @abc.abstractmethod
    def write_header(self, aliases: dict[str, str]) -> None:
        pass
```

### Реализация для CSV

```python
class CSVFile(IFile):
    def __init__(self, file_obj: Any, fieldnames: list[str]):
        self._writer = csv.DictWriter(file_obj, fieldnames=fieldnames, extrasaction="ignore")
        self._is_header_written = False

    def write_rows(self, rows: list[dict[str, Any]]) -> None:
        self._writer.writerows(rows)

    def write_header(self, aliases: dict[str, str] | None = None) -> None:
        if self._is_header_written:
            raise RuntimeError("Header already written.")
        self._writer.writerow(aliases) if aliases else self._writer.writeheader()
        self._is_header_written = True
```

### Интерфейс для менеджера записи

```python
class IFileWriterContextlib(abc.ABC):
    _file: IFile | None

    @contextlib.contextmanager
    @abc.abstractmethod
    def open(self):
        pass

    def write_rows(self, rows: list[dict[str, Any]]) -> None:
        if self._file is None:
            raise RuntimeError("File not initialized.")
        self._file.write_rows(rows)

    def write_header(self, aliases: dict[str, str] | None = None) -> None:
        if self._file is None:
            raise RuntimeError("File not initialized.")
        self._file.write_header(aliases)
```

### Локальная и облачная реализация

```python
class CSVLocalWriter(IFileWriterContextlib):
    def __init__(self, filepath: str, fieldnames: list[str]):
        self._filepath = filepath
        self._fieldnames = fieldnames

    @contextlib.contextmanager
    def open(self):
        print("Entering CSVLocalWriter")
        with open(self._filepath, "w") as local_file:
            self._file = CSVFile(local_file, self._fieldnames)
            yield self
        print("Exiting CSVLocalWriter")
```

```python
class CSVCloudWriter(IFileWriterContextlib):
    def __init__(self, uri: str, fieldnames: list[str], transport_params: dict[str, Any] | None = None):
        self._uri = uri
        self._fieldnames = fieldnames
        self._transport_params = transport_params

    @contextlib.contextmanager
    def open(self):
        print("Entering CSVCloudWriter")
        with smart_open.open(self._uri, "w", transport_params=self._transport_params) as cloud_file:
            self._file = CSVFile(cloud_file, self._fieldnames)
            yield self
        print("Exiting CSVCloudWriter")
```

Контекстный менеджер позволяет сократить использование `try/finally` и гарантирует корректное закрытие ресурса при ошибках.