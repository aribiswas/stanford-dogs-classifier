# Stanford Dogs Classifier
Identify dog breeds from images.

This project builds a classifier that can predict dog breeds from images. The classifier is trained on the [stanford_dogs](https://www.tensorflow.org/datasets/catalog/stanford_dogs) TensorFlow dataset.

![Image](./resources/sample_predictions.png)

## Contents
1. [Intallation and Setup](#Intallation-and-Setup)
2. Run
3. Tutorial
   * Download and preprocess dataset
   * Create a machine learning model
   * Train the model
4. Conclusion

## Installation and Setup

Clone this repository.
```
git clone https://github.com/aribiswas/stanford-dogs-classifier.git
```

The required packages for this project are specified in a ***requirements.txt*** file. You can create a virtual environment from the requirements file for this project. I use Anaconda for managing environments and the following steps are specific to it, but you can use any other environment manager of your choice. 

Miniconda is a lightweight version of Anaconda. You can download and install Miniconda from [here](https://docs.conda.io/en/latest/miniconda.html).

Once installed, navigate to the cloned repository and create a virtual environment:
```
conda create --name <YOUR-ENV-NAME> --file requirements.txt
```
Activate the environment and you are ready to run the project.
```
conda activate <YOUR-ENV-NAME>
```

## Run

The entry point script for this project is ***run.py***. You can use this script to identify dog breeds from image URLs.

From the project root execute the entry point script. This will prompt you to enter the URL of an image that you want to identify.
```
python run.py
Enter an url for the image, 0 to quit: 
```
Try this image: https://media.nature.com/lw800/magazine-assets/d41586-020-03053-2/d41586-020-03053-2_18533904.jpg

Altenately, you can provide many URLs as command line arguments.
```
python run.py --url <url1> <url2> ...
```

## Tutorial

The goal of this project is to build a classifier model that can accurately predict dog breeds from images. I use TensorFlow 2.0 and the [standford-dogs](https://www.tensorflow.org/datasets/catalog/stanford_dogs) TensorFlow Dataset to train the classifier. Following are the high level steps for reproducing my results:

### Download and preprocess dataset

The stanford-dogs dataset contains images of 120 dog breeds from around the world. The first step is to download the dataset in your project folder. The ***dataprocessor.py*** module contains the required utilities to download and preprocess the images. I will explain the high level steps in details.

Write a function to download the dataset. I use the *[tfds.load](https://www.tensorflow.org/datasets/api_docs/python/tfds/load)* function which downloads the dataset to a folder of your choice (data/tfds in the code).
```
import tensorflow as tf
import tensorflow_datasets as tfds

def load_dataset():
	(ds_train, ds_test), ds_info = tfds.load(
                                           'stanford_dogs', 
                                            split=['train', 'test'],
                                            shuffle_files=True,
                                            as_supervised=False,
                                            with_info=True,
                                            data_dir='data/tfds'
                                          )
        return ds_train, ds_test, ds_info
```
Use the *[tfds.show_examples()](https://www.tensorflow.org/datasets/api_docs/python/tfds/visualization/show_examples)* function to view a few sample images from the dataset.
```
ds_train, _, ds_info = load_dataset()
tfds.show_examples(ds_train, ds_info)
```
![Example_Image](./resources/examples.png)

Note that the images in the dataset have different sizes. In order to train an accurate model from these images, you need to resize the images to a reasonable size. An analysis of image dimension distributions is shown below. The red line shows the dimension (in pixels) with maximum frequency. We can infer that most images have width and height between 200-500 pixels.

To generate the histograms, you can use the  *analyze()* function in ***dataprocessor.py***.
```
import dataprocessor as proc
proc.analyze()
```
![Analyze_Image](./resources/analyze.png)

I chose to resize all images to (224,224,3) which is the dimension from the MobileNetV2 paper. In addition to resizing, I also cast the images to float, normalize the image between 0 and 1 and one hot encode the labels. 

Following is a function that performs the preprocessing on the dataset. 
```
def preprocess(data):
    image_size = (224, 224)
    num_labels = 120
    processed_image = data['image']
    label = data['label']
    processed_image = tf.cast(processed_image, tf.float32)
    processed_image = tf.image.resize(processed_image, image_size, method='nearest')
    processed_image = processed_image / 255.
    label = tf.one_hot(label, num_labels)
    return processed_image, label
```

Lets use the *preprocess* function to create a data pipeline for training and validation. The *[tfds.map()](https://www.tensorflow.org/api_docs/python/tf/data/Dataset#map)* method can be used to transform each entry i.e. (image,label) pair from the dataset using *preprocess()*. You can also use buffered prefetching to load data from disk without any blockage. More information on this [here](https://www.tensorflow.org/guide/data_performance).
```
def prepare(dataset):
    dataset = dataset.map(preprocess, num_parallel_calls=tf.data.AUTOTUNE)
    dataset = dataset.cache()
    dataset = dataset.batch(32)
    dataset = dataset.prefetch(buffer_size=tf.data.AUTOTUNE)
    return dataset
```

### Create a machine learning model

For this project I use a model architecture based on a pretrained [MobileNetV2](https://arxiv.org/abs/1801.04381) network to classify the images. Open *[model.py](https://github.com/aribiswas/stanford-dogs-classifier/blob/master/model.py)* to see other network architectures and feel free to experiment with them.

First, create a base model using *[tf.keras.applications.MobileNetV2](https://www.tensorflow.org/api_docs/python/tf/keras/applications/mobilenet_v2/MobileNetV2)* with trained weights. The include_top=False flag removes the fully connected layer at the top of the network. The base model will not be trainable and will be used to extract features from the images. The trainable part consists of a Dense layer that outputs the dog breed probabilities.

The code is shown below.
```
def mobilenet(image_shape, num_classes, lr=0.001):
    base_model = tf.keras.applications.MobileNetV2(input_shape=image_shape,
                                                   include_top=False,
                                                   weights='imagenet')
    base_model.trainable = False
    model = tf.keras.Sequential([
        base_model,
        tf.keras.layers.GlobalAveragePooling2D(),
        tf.keras.layers.Dense(num_classes, activation='softmax')
    ])

    model.compile(
        optimizer=tf.keras.optimizers.Adam(lr),
        loss='categorical_crossentropy',
        metrics=['accuracy', 'top_k_categorical_accuracy']
    )
    return model
```

### Train the model
Once the model is created, you are ready to train. The training utilities can be found in the ***trainer.py*** module. 

Load the dataset and prepare the train and test batches.
```
ds_train, ds_test, ds_info = load_dataset()

train_batches = prepare(ds_train)
test_batches  = prepare(ds_test)
```
Use the fit function to train the model. I also add a few callbacks for visualization (TensorBoard) and saving the model.
```
import datetime

log_dir = "logs/fit/" + datetime.datetime.now().strftime("%Y%m%d-%H%M%S")

# Create a callback for visualization in TensorBoard
# This will save training metrics inside the log_dir folder 
tensorboard_callback = tf.keras.callbacks.TensorBoard(log_dir=log_dir,
                                                      histogram_freq=1,
                                                      profile_batch=0 )
						      
# Create a callback that saves the model's weights
cp_callback = tf.keras.callbacks.ModelCheckpoint(filepath=checkpoint_path,
                                                 save_weights_only=True,
                                                 verbose=1 )

# Train
net.fit(
        train_batches,
        epochs=10,
        validation_data=test_batches,
        callbacks=[tensorboard_callback, cp_callback]
       )

# save the model weights
net.save(save_path + "/mobilenet_1")
```
You can visualize the training progress using TensorBoard. From a different terminal, execute the following command.
```
tensorboard --logdir logs/fit/<TIMESTAMP_FOLDER_NAME>
```
This will output something like this. Click on the link to see the training in a browser.
```
Serving TensorBoard on localhost; to expose to the network, use a proxy or pass --bind_all
TensorBoard 2.5.0 at http://localhost:6006/ (Press CTRL+C to quit)

```
The model achieved around 83% accuracy (training) and 75% accuracy (validation). The top-k categorical accuracy was significantly higher at 98% (training) and 95% (validation). The results are shown below.
![Results_image](./resources/train_result.png)

### Conclusion

Overall, the model achieved reasonable results after training. You can test the performance of the model using the ***run.py** script.
