.. _recogniz:

Программа для проверки кто сидит за компьютером
===============================================

Класс с описание пользователя
-----------------------------
Старался писать с применением ООП (но без фанатизма), поэтому содержится только один класс, в файле **user.py**::

	import pygame
	class User:
	    """
	    Base class for any user
	    """
	    def __init__(self, name, limit=20, sound='audio/alexey.wav'):
		self.name = name
		self.limit = limit
		self.sound = sound
		self.count = 0

	    def get_name(self):
		return self.name

	    def check_limit(self):
		if self.count >= self.limit:
		    return True
		else:
		    return False

	    def play(self):
		pygame.init()
		music = pygame.mixer.Sound(self.sound)
		music.play()

	    def sub(self, other):
		self.count -= other
		if self.count < 0:
		    self.count = 0

	    def add(self, other):
		self.count += other
		if self.count > self.limit:
		    self.count = self.limit

	    def __str__(self):
		return '{0}: {1}({2})'.format(self.name, self.count, self.limit)

Идея класса в том, что у каждого пользователя есть:
	- имя (name);
	- ограничение (limit);
	- звуковой файл который проигрывается что-бы выгнать его из-за компьютера (sound);
	- текущее кол-во минут, которое пользователь провел за компьютером (count). 

В классе предусмотренны методы для:
	- проверки не превышен ли лимит;
	- увеличение кол-ва проведенного времени (предполагается увеличение на 1);
	- уменьшение кол-ва проведенного времени (предполагается уменьшение на 2, чтобы быстрее восстанавдивалось);
	- не очень важные, вспомоготальные методы.

Оснобенно хочется отметитить конструкцию для воспроизведения звука, я с ней промучился больше чем с нейросеть. Сначала я хотел воспроизводить проигрывателем mplayer. 
Удивительно, но он работает плохо - если в это время звук уже проигрывается (запущена игра, например) то выдает ошибку - устройство занято. Насколья я понял, это из-за того,
что он пытается напрямую использовать alsa, но при этом не знает как правильно это сделать. Потом я пробовал другие проигрыватели, в общем выше лучший вариант. Звуковая подсистема в linux ubuntu из коробки к сожалению плохая.

Главный модуль
--------------
Именно в нем происходит получение снимка, распознование кто за компьютером и подача предупреждающего сигнала::

	from __future__ import absolute_import
	from __future__ import division
	from __future__ import print_function
	import os.path
	import cv2 # pip install opencv-python
	import datetime
	import time
	import logging
	import numpy as np
	import tensorflow as tf
	import subprocess
	import user



	def load_graph(model_file):
	    graph = tf.Graph()
	    graph_def = tf.GraphDef()

	    with open(model_file, "rb") as f:
		graph_def.ParseFromString(f.read())
	    with graph.as_default():
		tf.import_graph_def(graph_def)

	    return graph


	def read_tensor_from_image_file(file_name, input_height=299, input_width=299,
		                        input_mean=0, input_std=255):
	    input_name = "file_reader"
	    output_name = "normalized"
	    file_reader = tf.read_file(file_name, input_name)
	    if file_name.endswith(".png"):
		image_reader = tf.image.decode_png(file_reader, channels=3,
		                                   name='png_reader')
	    elif file_name.endswith(".gif"):
		image_reader = tf.squeeze(tf.image.decode_gif(file_reader,
		                                              name='gif_reader'))
	    elif file_name.endswith(".bmp"):
		image_reader = tf.image.decode_bmp(file_reader, name='bmp_reader')
	    else:
		image_reader = tf.image.decode_jpeg(file_reader, channels=3,
		                                    name='jpeg_reader')
	    float_caster = tf.cast(image_reader, tf.float32)
	    dims_expander = tf.expand_dims(float_caster, 0);
	    resized = tf.image.resize_bilinear(dims_expander, [input_height, input_width])
	    normalized = tf.divide(tf.subtract(resized, [input_mean]), [input_std])
	    sess = tf.Session()
	    result = sess.run(normalized)

	    return result


	def load_labels(label_file):
	    label = []
	    proto_as_ascii_lines = tf.gfile.GFile(label_file).readlines()
	    for l in proto_as_ascii_lines:
		label.append(l.rstrip())
	    return label


	if __name__ == "__main__":
	    # Initialization
	    logging.basicConfig(filename=os.path.join('logs', datetime.datetime.now().strftime('%Y-%m-%d %H:%M:%S.log')),
		                filemode='a', level=logging.DEBUG,
		                format='%(asctime)s %(levelname)s: %(message)s')
	    logging.info('Program is start')
	    errors_dir = "tf_files/errors"
	    model_file = "tf_files/retrained_graph.pb"
	    label_file = "tf_files/retrained_labels.txt"
	    input_height = 224
	    input_width = 224
	    input_mean = 128
	    input_std = 128
	    input_layer = "input"
	    output_layer = "final_result"
	    graph = load_graph(model_file)
	    # Create class for my users
	    users = (user.User('andrey', 10, 'audio/andrey.wav'),
		     user.User('alexey', 20, 'audio/alexey.wav'),
		     user.User('natasha', 20, 'audio/natasha.wav'))
	    logging.debug('Initialization completed')

	    while True:
		try:
		    log = list()
		    for user in users:
		        log.append(str(user))
		    logging.debug('; '.join(log))
		    # Get photo from camera
		    # Camera ON
		    cap = cv2.VideoCapture(0)
		    # Camera need must be hot, else photo will be dark
		    for i in range(30):
		        cap.read()
		    # Take photo
		    ret, frame = cap.read()
		    # Write to file
		    dt = datetime.datetime.now().strftime('%Y-%m-%d %H:%M:%S.jpg')
		    file_name = os.path.join('tf_files/shoot',dt)
		    cv2.imwrite(file_name, frame)
		    # Camera OFF
		    cap.release()
		    logging.debug('Take photo: ' + file_name)
		    # End camera work

		    # Tensor flow start
		    t = read_tensor_from_image_file(file_name,
		                                    input_height=input_height,
		                                    input_width=input_width,
		                                    input_mean=input_mean,
		                                    input_std=input_std)

		    input_name = "import/" + input_layer
		    output_name = "import/" + output_layer
		    input_operation = graph.get_operation_by_name(input_name)
		    output_operation = graph.get_operation_by_name(output_name)

		    with tf.Session(graph=graph) as sess:
		        start = time.time()
		        results = sess.run(output_operation.outputs[0],
		                           {input_operation.outputs[0]: t})
		        end = time.time()
		    results = np.squeeze(results)

		    top_k = results.argsort()[-5:][::-1]
		    labels = load_labels(label_file)
		    pl = ''
		    pr = 0
		    for i in top_k:
		        if results[i] > pr:
		            pr = results[i]
		            pl = labels[i]
		    logging.debug('About notebook %s (%s)' % (pl, pr))
		    if pr > 0.7:
		        os.remove(file_name)
		        for user in users:
		            if user.get_name() == pl:
		                # This is reload operation, don't change +=
		                user.add(1)
		                if user.check_limit():
		                    logging.info('For {0} have not time, will be warning'.format(user))
		                    user.play()
		            else:
		                user.sub(2)
		    else:
		        os.rename(file_name, os.path.join('tf_files/errors', dt))
		except:
		    logging.exception('')
		time.sleep(56)

У программы есть особенности:
	1. Используется ранее обученная модель, без нее не будет работать.
	2. Не всегда проходит успешное распознование, поэтому она считает, что пользователь за компьютером только если распознал с вероятностью более 0.7. Это минут, т.к. если сидят в двоем, то не срабатывает.
	3. Программа ведет лог файл.
	4. Фотографии которые не удалось распознать копируются в папку errors и потом могут быть использованы для тренировки сети.
	5. Программа выполняется в бесконечном цикле с паузой 56 сек (всего время 1 минута, в основном из-за прогрева камеры).
	
Логика заложена такая:
	1. Есть 4-ре образа который может быть распознан - никого нет (none) и 3 пользователя (alexey, natasha, andrey). 
	2. Если удалось распознать (с вероятностью 0.7), то найденному пользователю начисляется 1 минута, у других отнимается 2 минуты (да, восстановление в 2 раза быстрее, чем накопление).
	3. После каждого начисления проверяется не привышен ли лимит.
	


