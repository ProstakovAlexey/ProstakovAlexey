.. _neironet:

Подготовка и обучение нейросети
===============================
Весь проект буду вести в виртуальном окружении, т.к. наверняка будет море зависимостей. Создаю окружение, перехожу в папку окружения, активирую его ``source env-tf/bin/activate``.
Все дальнейшие действия будут выполнятся в окружении.

Подготовка изображений для обучения сети
----------------------------------------
Как я понял для работы сети необходимо подготовить большое кол-во изображений, разложенных по папкам в соответствии с объектами которые на них присутствуют.

Для этого я решил использовать видеокамеру ноута и простой скрипт на python, чтобы делать снимки раз в минуту. Далее эти снимки я вручную разложу по 4-м папкам.

Скрипт на python:: 

    import cv2
    import datetime
    # Включаем первую камеру
    cap = cv2.VideoCapture(0)
    # "Прогреваем" камеру, чтобы снимок не был тёмным
    for i in range(30):
        cap.read()
    # Делаем снимок
    ret, frame = cap.read()
    # Записываем в файл
    cv2.imwrite('/home/alexey/cam/' + datetime.datetime.now().strftime('%Y-%m-%d %H:%M:%S.jpg'), frame)
    # Отключаем камеру
    cap.release()

Запускается скрипт задание в cron::

      */1 * * * * python3 /home/alexey/get_cam.py

В результате работы скрипт в папке ``/home/alexey/cam`` собираются снимки с камеры. Предполагаю оставить скрипт в работе на 2-3 дня, а потом вручную рассортировать снимки. В результате я получил 4 папки:
        1. alexey - 171 файл, около 17Мб;
        2. andrey - 151 файл, около 15Мб;
        3. natasha - 78 файлов, 7.3Мб;
        4. none - 85 файлов, 8.2Мб.

Установка TensorFlow
--------------------
Моя установка отличается от указанной в статье, т.к. я ставлю на Ubuntu 16.04, а не на raspberry pi. Сделал следующее:
        1. Пускаю ``pip install tensorflow``. Если вы не уверены, что нужно делать окружение, то сейчас поймете, что это было верное решение.
        2. Клонирую репозитарий ``git clone https://github.com/googlecodelabs/tensorflow-for-poets-2.git`` в нем уже есть необходимые скрипты для создания сети.
        
Создание и обучение сети
------------------------
1. Создаю вот такую структуру в папке "для поэтов":: 

        $ls tensorflow-for-poets-2/tf_files/retraining/
        alexey  andrey  natasha  none

2. Копирую в папки подготовленные файлы, полученные снимки с камеры.

3. Запускаю обучение сети, все параметры взял из статьи, но оформил в виде скрипта, чтобы исключить ошибки при наборе::

        #!/bin/bash
        python3 -m scripts.retrain\
	--bottleneck_dir=tf_files/bottlenecks \
	--how_many_training_steps=300 \
	--model_dir=tf_files/models/ \
	--summaries_dir=tf_files/training_summaries/mobilenet_1.0_224 \
	--output_graph=tf_files/retrained_graph.pb \
	--output_labels=tf_files/retrained_labels.txt \
	--architecture=mobilenet_1.0_224 \
	--image_dir=tf_files/retraining

Примерно через минуту обучение закончилось, созданы два файла (они и понадобятся в будущем) retrained_labels.txt - файл меток, retrained_graph.pb - обученная сеть. Можно проверить результат.

Распознование
-------------
Ну наконец-то подошли к важному. Для проверки работы у поэтов есть скрипт ``label_image.py``. Для этого я подготовил несколько тестовых файлов. Запуск такой командой::

        $ python -m scripts.label_image --graph=tf_files/retrained_graph.pb --image=tf_files/tests/alexey/2.jpg 
        2019-01-03 17:25:28.054778: I tensorflow/core/platform/cpu_feature_guard.cc:141] Your CPU supports instructions that this TensorFlow binary was not compiled to use: AVX2 FMA
        Evaluation time (1-image): 0.293s
        alexey (score=0.99999)
        andrey (score=0.00001)
        natasha (score=0.00000)
        none (score=0.00000)
        
В общем на хороших кадрах результат отличный. Но не обольщайтесь, это примитивная сеть и до человека ей далеко. Были кадры, где двое одновременно - natasha и andrey. Тут ситуация такая::

        (env-tf) alexey@home:~/python-virtual-environments/tensorflow-for-poets-2$ python -m scripts.label_image --graph=tf_files/retrained_graph.pb --image=tf_files/tests/two/3.jpg 
        
        Evaluation time (1-image): 0.284s
        andrey (score=0.81522)
        natasha (score=0.18467)
        alexey (score=0.00012)
        none (score=0.00000)

        (env-tf) alexey@home:~/python-virtual-environments/tensorflow-for-poets-2$ python -m scripts.label_image --graph=tf_files/retrained_graph.pb --image=tf_files/tests/two/1.jpg 

        Evaluation time (1-image): 0.294s
        natasha (score=0.99966)
        alexey (score=0.00033)
        andrey (score=0.00001)
        none (score=0.00000)
        
Общий вывод - что-то получается, годится для опытной эксплуатации.
