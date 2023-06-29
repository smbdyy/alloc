## Кратко про мьютексы из freertos
Они реализуются с помощью семафора следующим образом:

```C++
SemaphoreHandle_t mutex = xSemaphoreCreateMutex(); // инициализация
if (mutex == nullptr) { /* не удалось инициализировать семафор */ }

// ...

if (xSemaphoreTake(mutex, portMAX_DELAY) == pdTRUE) { // семафор успешно захвачен
  
  // критическая секция

  xSemaphoreGive(mutex); // освобождаем семафор
}
else {
  // не удалось захватить семафор
  // ! освобождать его здесь не нужно
}
```

В данном случае мьютекс `pool_mutex` нужен для предотвращения одновременного доступа нескольких потоков к пулу памяти.

## Основные классы, структуры, методы

1. Класс `mem_pool`:
```C++
template<std::size_t POOL_SIZE = 1000, std::size_t CHUNK_SIZE = 10>
class mem_pool {
  // ...
};
```
Этот класс представляет собой пул памяти с фиксированным размером `POOL_SIZE` и фиксированным размером блока памяти `CHUNK_SIZE`. Здесь также присутствуют вспомогательные шаблонные структуры и методы для управления пулом памяти.

2. Константы и типы данных:
```C++
static constexpr std::size_t max_align = alignof(std::max_align_t);
static constexpr std::size_t chunks_cnt = div_with_round(POOL_SIZE, sizeof(chunk<CHUNK_SIZE>));

using chunk_type = std::aligned_storage_t<sizeof(chunk<CHUNK_SIZE>), alignof(chunk<CHUNK_SIZE>)>;
```
Константа `max_align` устанавливается равной выравниванию наиболее строго типизированного объекта, то есть `std::max_align_t`. Это значение представляет наибольшее выравнивание, которое может быть требуемым для выделения памяти в пуле.  
Константа `chunks_cnt` хранит количество блоков (`chunks`) памяти, которые содержит пул. Она рассчитывается путем деления размера пула (`POOL_SIZE`) на размер одного блока памяти (`sizeof(chunk<CHUNK_SIZE>)`). Функция `div_with_round` используется для округления результата вверх, чтобы учесть случаи, когда `POOL_SIZE` не делится нацело на размер блока памяти.    
Тип `chunk_type` представляет блок памяти фиксированного размера. Этот тип создается с использованием `std::aligned_storage_t`, который предоставляет хранилище памяти фиксированного размера с требуемым выравниванием. Размер блока памяти определяется как `sizeof(chunk<CHUNK_SIZE>)`, а выравнивание - `alignof(chunk<CHUNK_SIZE>)`.  

3. Структура `vec_all`:
```C++
template <typename T, std::size_t N>
struct vec_all {
  bool is_free{true};

  using value_type = T;

  template <typename U>
  struct rebind {
    using other = vec_all<U, N>;
  };

  // ...
};
```
Эта структура используется для представления вектора с фиксированным размером `N`, где каждый элемент отслеживает свободность соответствующего блока памяти.

4. Структура `mem_pool::pool`:
```C++
struct {
  // ...

  chunk_type chunk[chunks_cnt]{};
  std::vector<bool, vec_all<bool, chunks_cnt>> chunk_reserved;
} pool{};
```
В этой структуре хранятся фактические блоки памяти `chunk` и вектор `chunk_reserved`, который отслеживает свободность каждого блока памяти.

5. Класс `mem_allocator`:
```C++
template<class T, typename MEM_POOL>
class mem_allocator {
  // ...
};
```
Это сам аллокатор памяти, который использует пул памяти `MEM_POOL` для выделения и освобождения памяти для объектов типа `T`. Всё, что он делает в методах `allocate` и `deallocate` -- это вызывает соответствующие методы у `MEM_POOL`, а в методе `allocate` также проверяет, что память была успешно выделена и преобразует указатель на `void` в указатель на `T`.  
Также в нем реализованы операторы сравнения, parameterless-конструктор и copy-конструктор для соответствия стандарту.

## Методы для выделения и освобождения памяти
1. Метод `allocate`:
```C++
[[nodiscard]]
inline void* allocate(std::size_t const bytes_cnt, std::size_t const alignment = max_align) noexcept {
  if (xSemaphoreTake(pool_mutex, portMAX_DELAY) == pdTRUE) {
    if (bytes_cnt == 0) {
      xSemaphoreGive(pool_mutex);
      return &pool.chunk[0];
    }

    auto const &&is_free = [](auto const &a) { return !a; };
    auto const &&get_chunk = [this](auto const info_it) {
      std::size_t const index(std::distance(std::begin(pool.chunk_reserved), info_it));
      xSemaphoreGive(pool_mutex);
      return &pool.chunk[index];
    };

    for (auto it = std::begin(pool.chunk_reserved), end{std::end(pool.chunk_reserved)}; it != end;
         it = std::find_if(std::next(it), end, is_free)) {
      void* ptr{get_chunk(it)};
      std::size_t space{sizeof pool.chunk[0]};

      if (std::align(alignment, 0, ptr, space)) {
        if (bytes_cnt <= space) {
          *it = false;
          xSemaphoreGive(pool_mutex);
          return ptr;
        }

        decltype(std::distance(it, end)) const additional_chunks_needed(div_with_round(bytes_cnt - space, sizeof pool.chunk[0]));

        if (std::distance(it, end) >= additional_chunks_needed) {
          auto range_end = std::next(it, additional_chunks_needed + 1);

          if (std::all_of(std::next(it), range_end, is_free)) {
            std::fill(it, range_end, true);
            xSemaphoreGive(pool_mutex);
            return ptr;
          }
        }
      }
    }

    xSemaphoreGive(pool_mutex);
  } else {
    std::cerr << __PRETTY_FUNCTION__ << ": Failed to lock pool mutex" << std::endl;
    exit(EXIT_FAILURE);
  }

  return nullptr;
}
```

Метод`allocate` предназначен для выделения блока памяти заданного размера `bytes_cnt` с заданным выравниванием `alignment`. Здесь используется семафор `pool_mutex`, чтобы обеспечить потокобезопасность при доступе к пулу памяти.

* В начале метода происходит захват семафора `pool_mutex` с помощью функции `xSemaphoreTake`. Если захват прошел успешно, выполняется следующая логика:

* Если размер `bytes_cnt` равен нулю, то возвращается указатель на первый блок памяти `&pool.chunk[0]`.

* В противном случае, используются лямбда-функции `is_free` и `get_chunk`, чтобы проверить свободность блоков памяти и получить указатель на блок памяти соответственно.

* Затем выполняется цикл по вектору `pool.chunk_reserved`, где ищется подходящий блок памяти. Для этого используется функция `std::find_if`, которая ищет первый элемент, удовлетворяющий предикату `is_free`.

* Если найден блок памяти, используется функция `std::align`, чтобы проверить, можно ли выравнять блок памяти с указанным выравниванием. Если это возможно, проверяется достаточность места для выделения блока памяти.

* Если блок памяти достаточен для выделения, он помечается как занятый `(*it = false)`, семафор pool_mutex освобождается, и указатель на блок памяти возвращается.

* Если требуется дополнительное количество блоков памяти для выделения, проверяется, есть ли достаточное количество последовательных свободных блоков. Если такие блоки найдены, они помечаются как занятые, семафор `pool_mutex` освобождается, и указатель на первый блок в этой последовательности возвращается.

* Если не удалось найти подходящий блок памяти, семафор `pool_mutex` освобождается, и возвращается `nullptr`.

* Если захват семафора `pool_mutex` не удался, выводится сообщение об ошибке и программа завершается.

2. Метод `deallocate`:
```C++
inline void deallocate(void* const ptr, std::size_t const bytes_cnt, [[maybe_unused]]std::size_t const alignment = max_align) noexcept {
  if (ptr == nullptr || bytes_cnt == 0)
    return;

  std::size_t const first_chunk_index{get_chunk_index_by_ptr(ptr)};
  std::size_t const last_chunk_index{get_chunk_index_by_ptr(reinterpret_cast<std::uint8_t*>(ptr) + bytes_cnt)};
  std::size_t const chunks2deallocate{last_chunk_index - first_chunk_index + 1};

  if (xSemaphoreTake(pool_mutex, portMAX_DELAY) == pdTRUE) {
    std::fill_n(std::next(std::begin(pool.chunk_reserved), first_chunk_index), chunks2deallocate, false);
    xSemaphoreGive(pool_mutex);
  } else {
    std::cerr << __PRETTY_FUNCTION__ << ": Failed to lock pool mutex" << std::endl;
    exit(EXIT_FAILURE);
  }
}
```

Метод `deallocate` используется для освобождения ранее выделенного блока памяти по указанному указателю `ptr`. Снова используется семафор `pool_mutex` для обеспечения потокобезопасности.

* Если указатель `ptr` равен `nullptr` или размер `bytes_cnt` равен нулю, метод просто возвращает управление без выполнения дальнейших действий.

* В противном случае, метод определяет индексы первого и последнего блоков памяти, которые нужно освободить, с помощью функции `get_chunk_index_by_ptr`.

* Затем метод захватывает семафор pool_mutex и использует функцию `std::fill_n`, чтобы пометить соответствующие блоки памяти в векторе `pool.chunk_reserved` как свободные.

* После этого семафор `pool_mutex` освобождается.

* Если захват семафора `pool_mutex` не удался, выводится сообщение об ошибке и программа завершается.
