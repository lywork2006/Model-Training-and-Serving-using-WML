# Model Training and Serving using WML

## Preequisite

Install [IBM Cloud CLI](https://console.bluemix.net/docs/cli/reference/bluemix_cli/get_started.html#getting-started) and [Machine Learning Plugin]()

``` shell
bx plugin install ml_cli_plugin_osx
bx target -o ORG -s SPACE
``` 

## 1. Provision your WML instance


### 1.1 Create an instance of WML service and associated key using BX command line

``` shell
bx cf create-service pm-20 lite Animesh-WML
bx cf create-service-key Animesh-WML Animesh-WML-Key
``` 

### 1.2 Get your service credentials
``` shell
bx cf service-key Animesh-WML Animesh-WML-Key
```

### 1.3 Set the Machine Learning plugin it up with your creds obtained in step 2

``` shell
export ML_INSTANCE=11111111-aaaa-2222-bbbb-333333333333
export ML_USERNAME=44444444-cccc-5555-dddd-666666666666
export ML_PASSWORD=77777777-eeee-8888-ffff-999999999999
export ML_ENV=<url from credentials>
 ```
### 1.4 Test your WML instance

``` shell
AnimeshMacBook:~ animeshsingh$ bx ml list training-runs
Fetching the list of training runs ...
SI No   Name   guid   status   framework   version   submitted-at   

0 records found.
OK
List all training-runs successful
 ```
 
## 2. Provision an Object Storage Instance, and upload training data

Provision an [Object Storage instance](https://console.bluemix.net/catalog/services/cloud-object-storage), and then setup your [AWS S3 command line](https://aws.amazon.com/cli/). You then need to upload data in your Object storage. Here we are getting the data sets from [THE MNIST DATABASE of handwritten digits](http://yann.lecun.com/exdb/mnist/)

``` shell
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test

# Create your training data and result buckets
aws --endpoint-url=http://s3-api.us-geo.objectstorage.softlayer.net s3 mb <trainingDataBucket>
aws --endpoint-url=http://s3-api.us-geo.objectstorage.softlayer.net s3 mb <trainingResultBucket>

aws --endpoint-url=http://s3-api.us-geo.objectstorage.softlayer.net s3 cp t10k-labels-idx1-ubyte.gz s3://test-data-animesh/
aws --endpoint-url=http://s3-api.us-geo.objectstorage.softlayer.net s3 cp train-labels-idx1-ubyte.gz s3://test-data-animesh/
aws --endpoint-url=http://s3-api.us-geo.objectstorage.softlayer.net s3 cp t10k-images-idx3-ubyte.gz s3://test-data-animesh/
aws --endpoint-url=http://s3-api.us-geo.objectstorage.softlayer.net s3 cp train-images-idx3-ubyte.gz s3://test-data-animesh/

aws --endpoint-url=http://s3-api.us-geo.objectstorage.softlayer.net s3 ls s3://test-data-animesh
2018-03-10 00:14:49    1648877 t10k-images-idx3-ubyte.gz
2018-03-10 00:13:12       4542 t10k-labels-idx1-ubyte.gz
2018-03-10 00:15:22    9912422 train-images-idx3-ubyte.gz
2018-03-10 00:14:31      28881 train-labels-idx1-ubyte.gz
``` 

## 3. Create your Model Training Run
### 3.1 Create Deep Learning Model program and put them in a zip file

In this step we create a sample deep learning tensorflow program to train a model. For this, you must use the input_data.py and convolutional_network.py files, which you can find in the tf-model.zip file in this repository. This is for [THE MNIST DATABASE of handwritten digits](http://yann.lecun.com/exdb/mnist/)

In the convolutional_network.py file, there is one part which is important for IBM Watson Machine Learning service to score the model properly. The model should be trained to the RESULT_DIR/model directory after training is complete.

``` shell
zip tf-model.zip convolutional_network.py input_data.py

``` 

### 3.2 Create a Training Run Manifest File

Create a Training Run Manifest File. Please use the sample tf-train.yml from this repository. Make sure to point to your Object Storage instance
``` shell
model_definition:
  name: tf-mnist-showtest1
  author:
    name: DL Developer
    email: dl@example.com
  description: Simple MNIST model implemented in TF
  framework:
    name: tensorflow
    version: 1.2
  execution:
    command: python3 convolutional_network.py --trainImagesFile ${DATA_DIR}/train-images-idx3-ubyte.gz
      --trainLabelsFile ${DATA_DIR}/train-labels-idx1-ubyte.gz --testImagesFile ${DATA_DIR}/t10k-images-idx3-ubyte.gz
      --testLabelsFile ${DATA_DIR}/t10k-labels-idx1-ubyte.gz --learningRate 0.001
      --trainingIters 20000
    compute_configuration:
      name: small
training_data_reference:
  name: training_data_reference_name
  connection:
    endpoint_url: "https://s3-api.us-geo.objectstorage.service.networklayer.com"
    aws_access_key_id: "722432c254bc4eaa96e05897bf2779e2"
    aws_secret_access_key: "286965ac10ecd4de8b44306288c7f5a3e3cf81976a03075c"
  source:
    bucket: training-data
  type: s3
training_results_reference:
  name: training_results_reference_name
  connection:
    endpoint_url: "https://s3-api.us-geo.objectstorage.service.networklayer.com"
    aws_access_key_id: "722432c254bc4eaa96e05897bf2779e2"
    aws_secret_access_key: "286965ac10ecd4de8b44306288c7f5a3e3cf81976a03075c"
  target:
    bucket: training-results
  type: s3
``` 

## Submit, Monitor and Store a Training RUN (Note: need to download the BX deep learning add-ons)
``` shell
bx ml train tf-model.zip tf-train.yaml
bx ml list training-runs
bx ml show training-runs training-DOl4q2LkR
bx ml monitor training-runs training-DOl4q2LkR
``` 
## Deploy and Serve Models

