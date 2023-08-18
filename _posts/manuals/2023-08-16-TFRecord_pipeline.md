---
title: TFRecord dataset input pipeline
author: Jieun
date: 2023-08-16
layout: post
category: manuals
---

TensorBoard의 프로파일러 기능을 이용하면 특정 구간의 평균 step time과 그 디테일을 확인할 수 있다. 지금 하는 연구에서는 평균 step time에서 input time이 차지하는 비율을 확인하고, 스토리지 시스템과 I/O 라이브러리를 개선하여 이를 줄이는 것을 목표로 하고 있다. 

관련 논문은 *input pipeline stalls*라는 키워드로 검색하면 찾아볼 수 있다. 공통적으로 지적하는 것은 많은 딥러닝 워크로드에서는 (보통) 가속기에서 수행하는 학습(gradient computation)시간 못지 않게 input time이 길다는 사실이다. 구체적인 기준은 없으나, TensorBoard에서는 step time에서 input time이 차지하는 비율이 20%가 넘어가면 input-bound로 해석한다. 

논문 조사 중에 TensorFlow에서 지원하는 *TFRecord* 형식을 사용하면 데이터 세트를 훨씬 효율적으로 읽을 수 있다는, 흥미로운 사실을 알게 되었다. 데이터 세트를 TFRecord로 바꾸면서 학습 시간이 80% 이상 줄어들었다고 말하는 블로그 글도 보았다. TFRecord란, 실은 프로토콜 버퍼다. 샘플 각각을 '파일 이름' + '카테고리 이름' + '샘플의 이진 데이터'로 직렬화하여 저장하는 것이다. 

TFRecord를 사용하면 왜 읽기 성능이 좋아지는가? 공식 문서에서 언급하기로는 읽기 패턴이 선형적으로 바뀐다고 하는데, 디스크로부터 가져오는 데이터의 크기가 더 커져서 I/O 횟수가 줄어드는 것이 큰 이유로 보인다. 첫 번째 에포크에서 디스크 I/O는 샘플 개수만큼 발생한다. 그러나 TFRecord를 사용하면 여러 샘플이 한 파일에 저장되므로, 디스크 I/O는 배로 줄어들 것이다. 한 번에 많은 샘플을 가져오는 것.

다만 이를 실제로 확인하고자 했는데, 나는 기계학습/딥러닝 분야 문외한이고, TensorFlow도 써본 적 없어서 쉽지 않았다. 인터넷 상의 정보들은 TensorFlow 버전, 모델 종류에 따라 사용이 불가능하거나, 설명이 지나치게 적었다. 그래서 일주일을 악전고투하여 성공한 기록을 소상히 기록하여 공개한다.

## 환경
 * TensorFlow 2.1.2
 * Pre-trained MobileNet
 * ImageNet 2012 Validation Dataset (6.8GB)

## 데이터 세트를 여러 TFRecord로 저장하기
```python
# tfrecord_converter.py

import tensorflow as tf
import os
import sys
import re

def get_args(argv: list) -> dict:
    arg_parser = re.compile(r'--(?P<option_name>[a-zA-Z_]+)=(?P<value>[\S]+)')
    valid_options = [
        'dataset_path',
        'size_per_record',
        'output_path',
    ]
    settings = {
        # Default setting
        'dataset_path': 'images',
        'size_per_record': 52428800,    # 50MBytes
        'output_path': 'tfrecords'
    }

    for arg in argv:
        parsing_result = arg_parser.search(arg)
        if parsing_result is None:
            print('[ERR] Wrong option:', arg, '(Ignored)')
        else:
            option_name = parsing_result.group('option_name')
            option_value = parsing_result.group('value')
            if option_name not in valid_options:
                print('[ERR] Invalid option:', option_name, '(Ignored)')
            else:
                if option_name in ['size_per_record']:
                    option_value = int(option_value)
                settings[option_name] = option_value

    if settings['output_path'].endswith('/'):
        settings['output_path'] = settings['output_path'][:-1]
    
    return settings

def create_tfrecord(settings):
    if not os.path.isdir(settings['output_path']):
        os.mkdir(settings['output_path'], mode=0o777,)
    output_file_prefix = settings['output_path'] + '/out_'
    
    category_index = 0  # For one-hot encoding
    sample_counter = 0
    sample_size = 0     # Total sample size (Bytes)
    output_counter = 0
    
    writer = tf.io.TFRecordWriter(output_file_prefix + str(output_counter) + '.tfrecord')
    for category in os.listdir(settings['dataset_path']):
        category_dir = os.path.join(settings['dataset_path'], category)
        image_filenames = os.listdir(category_dir)
        
        for image_filename in image_filenames:
            image_path = os.path.join(category_dir, image_filename)
            image = tf.io.read_file(image_path)
            sample_counter += 1
            sample_size += os.path.getsize(image_path)

            example = tf.train.Example(features=tf.train.Features(feature={
                'name': tf.train.Feature(bytes_list=tf.train.BytesList(value=[image_filename.encode('utf-8')])),
                'category': tf.train.Feature(int64_list=tf.train.Int64List(value=[category_index])),
                'data': tf.train.Feature(bytes_list=tf.train.BytesList(value=[image.numpy()]))
            }))
            
            writer.write(example.SerializeToString())

            if sample_size >= settings['size_per_record']:
                writer.close()
                print('TFRecord file ' + output_file_prefix +  str(output_counter) + '.tfrecord (contains ' + str(sample_counter) + ' samples) created.')

                output_counter += 1
                sample_counter = 0
                sample_size = 0
                writer = tf.io.TFRecordWriter(output_file_prefix + str(output_counter) + '.tfrecord')

        category_index += 1
    
    if sample_counter != 0:
        writer.close()
        print('TFRecord file ' + output_file_prefix + str(output_counter) + '.tfrecord (contains ' + str(sample_counter) + ' samples) created.')

def main():
    settings = get_args(sys.argv[1:])
    create_tfrecord(settings)

main()
```

바로 가져다가 적용할 수 있는 스크립트를 우선 공개한다. 사용 방법은 다음과 같다.

```shell
python3 tfrecord_converter.py --dataset_path=images --size_per_record=16777216 --output_path=tfrecords
```
 * `--dataset_path`: 데이터 세트의 경로. 해당 경로에는 샘플이 카테고리 별로 분류되어 있다고 가정한다.
    ```
    + images
      + n01440764
        - ILSVRC2012_val_00000293.JPEG
        - ILSVRC2012_val_00002138.JPEG
        ...
      + n01443537
        - ILSVRC2012_val_00000236.JPEG
        - ILSVRC2012_val_00000262.JPEG
        ...
      ...
    ```
 * `--size_per_record`: 변환된 TFRecord의 개별 크기. 바이트 단위로, 정확하게 명시된 크기의 TFRecord를 생성하는 것은 아니다.
 * `--output_path`: 변환된 TFRecord 파일들을 저장할 디렉토리. 없는 경우 생성한다.

TensorFlow 라이브러리를 이용하는 것 외에는 여타 직렬화와 과정은 같다. 그러나, **카테고리를 [0, N)의 정수로 바꾸는 것이 중요하다.** TensorFlow에서 제공하는 `tf.one_hot()` 메소드는 `indices`와 `depth`를 인자로 받는다. 그런데, `indices`는 정수 또는 정수 배열이어야 한다. 그리고 해당 메소드는 `indices` 값이 `depth`를 넘어서는 경우에는 모든 원소가 0인 one-hot 벡터를 반환한다.

```python
tf.one_hot(10, depth=5)
```
예를 들어 위의 스크립트는,

```python
[0, 0, 0, 0, 0]
```
을 반환한다. 카테고리를 의미하는 정수로 `depth` 이상인 값을 할당하면, 해당 샘플은 정보값을 잃는 셈이다. (어떻게 이게 아무런 warning 없이 실행되는 것인지…)

## 여러 TFRecord를 읽어 모델 학습시키기
```python
import tensorflow as tf

def parse_tfrecord(serialized_example):
    feature_description = {
        'name': tf.io.FixedLenFeature([], tf.string),
        'category': tf.io.FixedLenFeature([], tf.int64),
        'data': tf.io.FixedLenFeature([], tf.string)
    }
    example = tf.io.parse_single_example(serialized_example, feature_description)
    
    image = tf.io.decode_jpeg(example['data'], channels=3)
    image = tf.image.resize(image, [224, 224])
    image = (image / 127.5) - 1
    label = tf.one_hot(example['category'], depth=1000)
    
    return image, label

dataset = tf.data.TFRecordDataset(tfrecord_list)    # tfrecord_list: paths of tfrecords
dataset = dataset.map(parse_tfrecord)
dataset = dataset.shuffle(buffer_size=10240)
dataset = dataset.batch(32)                         # Batch size = 32

base_model = tf.keras.applications.MobileNet(weights='imagenet', include_top=False, input_shape=(224, 224, 3))
x = base_model.output
x = tf.keras.layers.GlobalAveragePooling2D()(x)
x = tf.keras.layers.Dense(1000, activation='softmax')(x)
model = tf.keras.Model(inputs=base_model.input, outputs=x)

model.compile(optimizer=tf.keras.optimizers.Adam(learning_rate=0.0001), loss='categorical_crossentropy', metrics=['accuracy'])

model.fit(dataset, epochs=10)
```

`parse_tfrecord`가 반환하는 텐서의 형태는 모델에 따라 달라진다. 여기서는 MobileNet에 맞게 이미지 데이터와 라벨로 이루어진 텐서를 반환한다.

MobileNet의 채널 정규화 범위 [-1, 1]을 맞춰주었으며, [0, 1000) 범위의 정수 값을 갖는 카테고리를 one-hot 벡터로 변환했다.
