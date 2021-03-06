Лабораторная работа №2
====
# Цель лабораторной работы
Обучить нейронную сеть с использованием техники обучения Transfer Learning.
Фактически, в данной работе предобученная сеть будет дообучать классификатор для нашего датасета. 

# Примечания
За основу взять сеть EfficientNet-B0. Так как параметр RESIZE_TO в нашем случае равен 224, используется именно EfficientNet-B0.
Информация об этом можно найти [здесь](https://keras.io/examples/vision/image_classification_efficientnet_fine_tuning/)

# 1. Обучить нейронную сеть EfficientNet-B0 (случайное начальное приближение) для решения задачи классификации изображений Food-101
**Архитектура:**
 
 Основные параметры взяты из [примера](https://github.com/AlexanderSoroka/CNN-food-101 "GitHub")
 
* По традици описание начнём с размерности входного изображения: 
```
inputs = tf.keras.Input(shape=(RESIZE_TO, RESIZE_TO, 3))
```

**Так как в данном случае, задача стоит использовать случайное приближение, 
использование сети  EfficientNet-B0 будет происходить с такими параметрами:**
* include_top=True - использование верхнего слоя сети EfficientNet-B0 (классификатора)
* weights=None - параметр случайного пначального приближения
* classes = NUM_CLASSES - количество классов в классификаторе, у нас это 101 класс
```
outputs = EfficientNetB0(include_top=True, weights=None, classes = NUM_CLASSES)(inputs)
```

 ### Графики обучения со случайным начальным приближением (BATCH_SIZE = 64):
 
* Синяя линия - валидация
* Оранжевая линия - обучение

 ***График точности:*** 
<img src="./stock_logs_64_batch/Графики/epoch_categorical_accuracy.svg">

 ***График потерь:*** 
<img src="./stock_logs_64_batch/Графики/epoch_loss.svg">

### Предворительный вывод:

Максимальная точность обучения со случайным начальным приближением в нашем случае равна 35%. Причём это значение достигается примерно на 3 эпохе, после чего график убывает, что даёт право утверждать об слабом обучении или вовсе его отсутствии в нашей сети. Это так же подтверждается графиком потерь.

# 2. Использование техники обучения Transfer Learning. Дообучить классификатор нейронной сети EfficientNet-B0 для решения задачи классификации изображений Food-101
**Архитектура:** 

**В данном случае также используется сеть EfficientNet-B0, но с иными от первого эксперимента параметрами:**
* include_top=False - так как в данном случае нам не нужен базовый классификатор EfficientNet
* input_tensor - параметр, отвечающий за вход изображей
* weights='imagenet' - Использование начального приближения на основе предобученности сети EfficientNet
* pooling='avg' - выбирает усреднённое значение на последнём свёрточном слое 
* (так же можно использовать со значением 'max' - максимальное значение на том же слое)

* Так же будет проведён опыт с параметром drop_connect_rate=0.4 - разрешающий параметр
(по умолчанию drop_connect_rate имеет значение 0.2)
* Нормализация уже определена внутри сети EfficientNet-B0

```
x = EfficientNetB0(include_top=False, input_tensor=inputs, pooling='avg', weights='imagenet')(inputs) 
```

* Отключение атрибута обучения, так как в нашем случае необходимо дообучить только наш классификатор:
```
x.trainable = False
```

* Преобразование многомерного тензора в одномерный
```
 x = tf.keras.layers.Flatten()(x.output)
```

* Полносвязный Dense слой с 101 выходом и функцией активации softmax, которая определяет к какой категории и с какой вероятностью относится входное изображение:
```
outputs = tf.keras.layers.Dense(NUM_CLASSES, activation=tf.keras.activations.softmax)(x)
```

* Для корректной работы Imagenet необходимо использовать тип unit8
```
example['image'] = tf.image.convert_image_dtype(example['image'], dtype=tf.uint8)
```

 ### Графики обучения для предобученной нейронной сети (BATCH_SIZE = 64):
 
* Синяя линия - на валидации
* Оранжевая линия - на обучении

 ***График точности:*** 
<img src="./modif_logs_64_batch/Графики/epoch_categorical_accuracy.svg">

 ***График потерь::*** 
<img src="./modif_logs_64_batch/Графики/epoch_loss.svg">

 ### Эксперимент с параметром drop_connect_rate (BATCH_SIZE = 128)
 
* Синяя линия - на валидации при drop_connect_rate = 0.2
* Оранжевая линия - на обучении при drop_connect_rate = 0.2

* Голубая линия - на валидации при drop_connect_rate = 0.4
* Красная линия - на обучении при drop_connect_rate = 0.4


 ***График метрики точности:*** 
<img src="./modif_logs_128_batch/Графики/epoch_categorical_accuracy.svg">

 ***График функции потерь::*** 
<img src="./modif_logs_128_batch/Графики/epoch_loss.svg">

### Вывод:

После перехода обучения на технику Transfer Learning графика метрики показывает большую точность (порядка 84% на обучающей выборке и 66% на валидационной), график потерь 
так же свидетельствует этому. В целом, можно утверждать, наша сеть не только обучается, но и переобучается. Для избежания этого, можно уменьшить время обучения (для этого достаточно порядка 5-10 эпох в нашем случае), сделать более слабую модель, а так же с этим позволяет бороться параметр drop_connect_rate. Как видно из последнего графика, уменьшилось переобучение, но так же упала и точность, это следствие того, что нашей сети стало сложнее обучаться.
В общем в первом случае (случайное приближение) сеть фактически не обучается, так достигнув максимальной точности примерно на 3 эпохе, точность начинает стремительно падать. Во же втором случае (Transfer Learning), как уже описывалось выше, наблюдается переобучение. Тем самым, техника Transfer Learning улучшило максимальное значение точности примерно на 50% на обучающей выборке и на 30% на валидационной выборке.
