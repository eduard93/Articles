# Accessing GPU from InterSystems IRIS container

Prereqs:

1. На хост-машине установлена ОС Ubuntu 18.04 (
2. На хост машине установлен Docker последней стабильной версии (минимум - Docker 19.03 либо Docker 20.10)
3. На хост-машине доступна видеокарта с поддержкой cuda 10.1 или 11.0 (Архитектура >= Kepler (или compute capability 3.0))
4. На хост-машине установлены драйверы для видеокарты версии >= 418.81.07 (желательно самые новые)
5. На хост машине установлен NVIDIA Container Toolkit:
   - https://unrealcontainers.com/docs/concepts/nvidia-docker
   - https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/instal...

Тест:

docker run --rm --gpus all nvidia/cuda:11.4.0-base nvidia-smi

Билдим контейнер https://github.com/intersystems-community/PythonGateway/blob/master/Dock...

Тестируем:

docker pull store/intersystems/iris-community:2021.1.0.215.0
import sys
sys.argv=['']
import tensorflow as tf
tf.test.is_built_with_cuda()
tf.config.list_physical_devices('GPU')
tf.debugging.set_log_device_placement(True)
a = tf.constant([[1.0, 2.0, 3.0], [4.0, 5.0, 6.0]])
b = tf.constant([[1.0, 2.0], [3.0, 4.0], [5.0, 6.0]])
c = tf.matmul(a, b)
print(c)

 

2021-08-10 14:17:27.903896: I tensorflow/core/common_runtime/eager/execute.cc:733] Executing op MatMul in device /job:localhost/replica:0/task:0/device:GPU:0

 

EMPY

FROM arti.iscinternal.com/intersystems/iris:2021.1.0PYTHON.330.0

RUN $ISC_PACKAGE_INSTALLDIR/bin/irispython -m pip install --no-cache-dir ${TF_PACKAGE}${TF_PACKAGE_VERSION:+==${TF_PACKAGE_VERSION}}
