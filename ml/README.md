# Model training for the Metal Nut Data Set
## Metal Nut Data Set
- Credits to https://www.mvtec.com/company/research/datasets
- See also: https://www.mvtec.com/company/research/datasets/mvtec-ad

### ATTRIBUTION
Paul Bergmann, Michael Fauser, David Sattlegger, Carsten Steger. MVTec AD - A Comprehensive Real-World Dataset for Unsupervised Anomaly Detection; in: IEEE Conference on Computer Vision and Pattern Recognition (CVPR), June 2019

### LICENSE
The data is released under the Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International License (CC BY-NC-SA 4.0). For using the data in a way that falls under the commercial use clause of the license, please contact us via the form below.

# Prerequisites

- Clone this repo into you home directory
- OpenShift Cluster with a GPU


# Convert Data Set to Darknet Yolo format

## Download the Data Set

Download the Metal Nut Data Set on you computer in `manuela-visual-inspection/ml/data`. 

```
cd ~/manuela-visual-inspection/ml/data

curl -u guest:GU.205dldo ftp://ftp.softronics.ch/mvtec_anomaly_detection/metal_nut.tar.xz -o metal_nut.tar.xz

tar xf metal_nut.tar.xz && rm -f metal_nut.tar.xz

ls -1d metal_nut/*
metal_nut/ground_truth
metal_nut/license.txt
metal_nut/readme.txt
metal_nut/test
metal_nut/train
```


## Create Yolo annotations files 

The data set contains also mask images which show the location of the anomalies. The script `generate-yolo-conf.py` creates Yolo annotaions files for all images using the mask files and detecting the contours in mask files for `scratch` and `bent` images in `manuela-visual-inspection/ml/darknet/data/metal_yolo`.


Switch to `~/manuela-visual-inspection/ml/scripts` and run `generate-yolo-conf.py` 

```
python generate-yolo-conf.py
```
```
ls -1 ../darknet/data/metal_yolo/
bent-000.png
bent-000.txt
bent-001.png
bent-001.txt
...
```


## Package the data for training

**Copy darknet config files**


Switch to `~/manuela-visual-inspection/ml/yolo-cfg` and copy files 

```
cd ~/manuela-visual-inspection/ml/yolo-cfg
cp metal-data.data ../darknet/data/metal-data.data
cp yolov4-custom-metal-ocp.cfg ../darknet/data/yolov4-custom-metal.cfg

```

**zip images and annotations, push to git**

```
cd ../darknet/
zip -r data.zip data
git add data.zip
git commit -m "new data.zip"
git push
```


# Yolo Model training on OpenShift using Darknet

The Yolov4 model training is performed with [Darknet](https://github.com/pjreddie/darknet). Darknet is an open source neural network framework written in C and CUDA. It is fast, easy to install, and supports CPU and GPU computation.

For a good Yolov4 introdction with Darknet plwase watch [YOLOv4 in the CLOUD: Install and Run Object Detector](https://www.youtube.com/watch?v=mKAEGSxwOAY)

Running the training on CPUs or not that powerfull GPUs takes very long. Therefore, let's use a Kubernetes job for the training.


## Build Darknet image (GPU)

First create a GPU enabled darknet images:

```
cd ~/manuela-visual-inspection/ml
oc project manuela-visual-inspection

oc apply -f manifests/darknet-gpu-is.yaml
oc apply -f manifests/darknet-gpu-bc.yaml
```

Watch the build logs:
```
oc logs bc/darknet-gpu
...
Successfully pushed image-registry.openshift-image-registry.svc:5000/manuela-visual-inspection/darknet-gpu@sha256:084350af727e3922ae1ee2ce3c44203715495d8b8a935a6e479146c33d7ce376
Push successful
```


## Run Training

```
oc apply -f manifests/darknet-gpu-pvc.yaml
oc apply -f manifests/darknet-metal-gpu-job.yaml
```

Note, trained weights are stored on a persistance volume `darknet-gpu-volume`, but you need to copy/save it to your computer manually.

## Save the trained model on you local computer


```
oc apply -f manifests/darknet-metal-depl.yaml

oc get pods
NAME                             READY   STATUS             RESTARTS   AGE
darknet-metal-777f88b5d8-wlnfb   1/1     Running            0          3d5h

oc cp darknet-metal-777f88b5d8-wlnfb:/mnt/darknet/backup ~/manuela-visual-inspection/ml/data/weights


```

## Optionally, Check Model Mean Average Precision (mAP) manually

Find out the mAP of your model after the training. Run the following command on any of the saved weights from the training to see the mAP value for that specific weight's file. 
Note, you have to install darknet on your computer first.

```
cd darknet
```

**1000.weights**:
```
darknet detector map data/metal_data_ocp.data data/yolov4-custom-metal.cfg ../data/weights/yolov4-custom-metal_1000.weights

class_id = 0, name = scratch, ap = 100.00%       (TP = 1, FP = 3) 
class_id = 1, name = bent, ap = 88.33%           (TP = 6, FP = 3) 

 for conf_thresh = 0.25, precision = 0.54, recall = 1.00, F1-score = 0.70 
 for conf_thresh = 0.25, TP = 7, FP = 6, FN = 0, average IoU = 36.56 % 

 IoU threshold = 50 %, used Area-Under-Curve for each unique Recall 
 mean average precision (mAP@0.50) = 0.941667, or 94.17 % 
```


**2000.weights**:
```
darknet detector map data/metal_data_ocp.data data/yolov4-custom-metal.cfg ../data/weights/yolov4-custom-metal_2000.weights

class_id = 0, name = scratch, ap = 100.00%       (TP = 1, FP = 3) 
class_id = 1, name = bent, ap = 100.00%          (TP = 6, FP = 0) 

 for conf_thresh = 0.25, precision = 0.70, recall = 1.00, F1-score = 0.82 
 for conf_thresh = 0.25, TP = 7, FP = 3, FN = 0, average IoU = 56.24 % 

 IoU threshold = 50 %, used Area-Under-Curve for each unique Recall 
 mean average precision (mAP@0.50) = 1.000000, or 100.00 % 
```

**final.weights**:
```
darknet detector map data/metal_data_ocp.data data/yolov4-custom-metal.cfg ../data/weights/yolov4-custom-metal_final.weights

class_id = 0, name = scratch, ap = 100.00%       (TP = 1, FP = 1) 
class_id = 1, name = bent, ap = 100.00%          (TP = 6, FP = 0) 

 for conf_thresh = 0.25, precision = 0.88, recall = 1.00, F1-score = 0.93 
 for conf_thresh = 0.25, TP = 7, FP = 1, FN = 0, average IoU = 71.37 % 

 IoU threshold = 50 %, used Area-Under-Curve for each unique Recall 
 mean average precision (mAP@0.50) = 1.000000, or 100.00 % 
 ```


 ## Optionally, package model in tar file

This step is optionlay because we will content to and use a Tensorflow model

```
tar -C data/weights/ -cvf data/release/model.tar yolov4-custom-metal_final.weights
tar -C yolo-cfg/ -rvf data/release/model.tar yolov4-custom-metal-test.cfg
tar -C darknet/data/metal_yolo/ -rvf data/release/model.tar classes.txt

tar tvf data/release/model.tar
```

Optionally, create release on GitHub



# Convert Darknet Yolo to Tensorflow

## Convert model

```
cd ~
git clone https://github.com/theAIGuysCode/tensorflow-yolov4-tflite.git
cd tensorflow-yolov4-tflite

```

```
python save_model.py --weights ~/manuela-visual-inspection/ml/data/weights/yolov4-custom-metal_final.weights --output ~/manuela-visual-inspection/ml/data/tf-model --input_size 416 --model yolov4

```


## Package TF model in tar file

```
cd ml
tar -C data/ -cvf data/release/tf-model.tar tf-model/
tar -C darknet/data/metal_yolo/ -rvf data/release/tf-model.tar classes.txt
```

Create a release on GitHub with the `tf-model.tar` file so that is can be used in the image-processor for predictions.
E.g. https://github.com/stefan-bergstein/manuela-visual-inspection/releases/download/v0.1-alpha-tf/tf-model.tar

