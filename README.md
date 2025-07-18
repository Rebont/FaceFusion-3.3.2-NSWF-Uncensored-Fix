# Отключение цензуры и проверки файлов в FaceFusion 3.3.2
**Путь к файлам:** pinokio\api\facefusion-pinokio.git\facefusion\facefusion

## Введение

FaceFusion 3.3.2 включает в себя две защитные системы, которые иногда мешают работе:
1. Модуль проверки NSFW-контента (цензура)
2. Система проверки целостности файлов

Эта инструкция поможет вам отключить обе защиты, изменив минимальное количество кода.

## 1. Отключение модуля проверки NSFW-контента

### Файл: content_analyser.py
**Путь:** `pinokio\api\facefusion-pinokio.git\facefusion\facefusion\content_analyser.py`

#### Функция `detect_nsfw`

**До изменения:**
```python
def detect_nsfw(vision_frame : VisionFrame) -> bool:
    return detect_with_nsfw_1(vision_frame) or detect_with_nsfw_2(vision_frame) or detect_with_nsfw_3(vision_frame)
```

**После изменения:**
```python
def detect_nsfw(vision_frame : VisionFrame) -> bool:
    # Отключаем проверку всех NSFW моделей
    return False
```

#### Функция `analyse_frame`

**До изменения:**
```python
def analyse_frame(vision_frame : VisionFrame) -> bool:
    return detect_nsfw(vision_frame)
```

**После изменения:**
```python
def analyse_frame(vision_frame : VisionFrame) -> bool:
    return False
```

#### Функция `analyse_image`

**До изменения:**
```python
@lru_cache(maxsize = None)
def analyse_image(image_path : str) -> bool:
    vision_frame = read_image(image_path)
    return detect_nsfw(vision_frame)
```

**После изменения:**
```python
@lru_cache(maxsize = None)
def analyse_image(image_path : str) -> bool:
    vision_frame = read_image(image_path)
    return False
```

#### Функция `analyse_video`

**До изменения:**
```python
@lru_cache(maxsize = None)
def analyse_video(video_path : str, trim_frame_start : int, trim_frame_end : int) -> bool:
    video_fps = detect_video_fps(video_path)
    frame_range = range(trim_frame_start, trim_frame_end)
    
    with tqdm(total = len(frame_range), desc = wording.get('analysing'), unit = 'frame', ascii = ' =', disable = state_manager.get_item('log_level') in [ 'warn', 'error' ]) as progress:
        for frame_number in frame_range:
            if frame_number % int(video_fps) == 0:
                vision_frame = read_video_frame(video_path, frame_number)
                if vision_frame is None or detect_nsfw(vision_frame):
                    return True
                progress.update()

    return False
```

**После изменения:**
```python
@lru_cache(maxsize = None)
def analyse_video(video_path : str, trim_frame_start : int, trim_frame_end : int) -> bool:
    video_fps = detect_video_fps(video_path)
    frame_range = range(trim_frame_start, trim_frame_end)
    
    with tqdm(total = len(frame_range), desc = wording.get('analysing'), unit = 'frame', ascii = ' =', disable = state_manager.get_item('log_level') in [ 'warn', 'error' ]) as progress:
        for frame_number in frame_range:
            if frame_number % int(video_fps) == 0:
                progress.update()

    return False
```

#### Исправление ошибки в функции `detect_with_nsfw_1`

В файле могут возникать ошибки, связанные с типом bool. Если вы столкнулись с ошибкой `__getitem__`, замените функцию:

**До изменения:**
```python
def detect_with_nsfw_1(vision_frame : VisionFrame) -> bool:
    detect_vision_frame = prepare_detect_frame(vision_frame, 'nsfw_1')
    detection = forward_nsfw(detect_vision_frame, 'nsfw_1')
    detection_score = numpy.max(numpy.amax(detection[:, 4:], axis = 1))
    return bool(detection_score > 0.0)[0]
```

**После изменения:**
```python
def detect_with_nsfw_1(vision_frame : VisionFrame) -> bool:
    detect_vision_frame = prepare_detect_frame(vision_frame, 'nsfw_1')
    detection = forward_nsfw(detect_vision_frame, 'nsfw_1')
    detection_score = numpy.max(numpy.amax(detection[:, 4:], axis = 1))
    
    detection_score_result = detection_score > 0.0
    if isinstance(detection_score_result, numpy.ndarray):
        return bool(detection_score_result[0])
    return bool(detection_score_result)
```

## 2. Отключение проверки хешей файлов

### Файл: core.py
**Путь:** `pinokio\api\facefusion-pinokio.git\facefusion\facefusion\core.py`

#### Функция `common_pre_check`

**До изменения:**
```python
def common_pre_check() -> bool:
    common_modules =\
    [
        content_analyser,
        face_classifier,
        face_detector,
        face_landmarker,
        face_masker,
        face_recognizer,
        voice_extractor
    ]
    
    is_valid = hash_helper.validate_content_analyser_hash()
    
    return all(module.pre_check() for module in common_modules) and is_valid
```

**После изменения:**
```python
def common_pre_check() -> bool:
    common_modules =\
    [
        content_analyser,
        face_classifier,
        face_detector,
        face_landmarker,
        face_masker,
        face_recognizer,
        voice_extractor
    ]
    
    # Пропускаем проверку хеша, всегда возвращая True
    is_valid = True
    
    return all(module.pre_check() for module in common_modules) and is_valid
```

## Советы и решение проблем

### Перед внесением изменений

1. **Создайте резервные копии файлов**
   - Сохраните оригиналы файлов перед модификацией
   - При обновлении программы повторите изменения

2. **Если возникают ошибки импорта модулей**
   - Проверьте путь установки
   - Используйте Python 3.10 (рекомендуется)

### После изменений

После внесенных изменений FaceFusion будет работать без цензуры NSFW-контента и без проверки целостности файлов. Это позволит:

- Обрабатывать любые изображения и видео
- Вносить собственные изменения в код
- Экспериментировать с параметрами и настройками

Эти модификации предназначены для образовательных целей и личного использования. Используйте на свой страх и риск.

