# Retina blood vessel segmentation

![](test/test_Original_GroundTruth_Prediction3.png)

This repository contains the implementation of a convolutional neural network used to segment blood vessels in retina fundus images. This is a binary classification task: the neural network predicts if each pixel in the fundus image is either a vessel or not.  


## Methods
Before training, the 20 images of the DRIVE training datasets are pre-processed with the following transformations:
- Gray-scale conversion
- Standardization
- Contrast-limited adaptive histogram equalization (CLAHE)
- Gamma adjustment

The training of the neural network is performed on sub-images (patches) of the pre-processed full images. Each patch, of dimension 48x48, is obtained by randomly selecting its center inside the full image. Also the patches partially or completely outside the Field Of View (FOV) are selected, in this way the neural network learns how to discriminate the FOV border from blood vessels.  
A set of 190000 patches is obtained by randomly extracting 9500 patches in each of the 20 DRIVE training images. Although the patches overlap, i.e. different patches may contain same part of the original images, no further data augmentation is performed. The first 90% of the dataset is used for training (171000 patches), while the last 10% is used for validation (19000 patches).

The neural network architecture is derived from the *U-net* architecture (see the [paper](https://arxiv.org/pdf/1505.04597.pdf)).
The loss function is the cross-entropy and the Adam is employed for optimization. The activation function after each convolutional layer is ReLU, and a dropout of 0.5 (default) is used between two consecutive convolutional layers. The output activation function is sigmoid. The output pixel is a one-channel scalar which implies the probability if the pixel is vessel.
Training is performed for 100 epochs, with a batch size of 1024 patches. Using a GeForce GTX 1080ti GPU the training lasts for about 4 hours.


## Results on DRIVE database
Testing is performed with the 20 images of the DRIVE testing dataset. Only the pixels belonging to the FOV are considered. The FOV is identified with the masks included in the DRIVE database.

The results reported in the `./test190000` folder are referred to the trained model which reported the minimum validation loss. The `./test190000` folder includes:
- Model:
  - `model_best.pth.tar` weights of the model which reported the minimum validation loss, as HDF5 file
  - `checkpoint.pth.tar`  weights of the model at last epoch (150th), as HDF5 file
- Experiment results:
  - `performances.txt` summary of the test results, including the confusion matrix
  - `Precision_recall.png` the precision-recall plot and the corresponding Area Under the Curve (AUC)
  - `ROC.png` the Receiver Operating Characteristic (ROC) curve and the corresponding AUC
  - `all_*.png` the 20 images of the pre-processed originals, ground truth and predictions relative to the DRIVE testing dataset
  - `sample_input_*.png` sample of 40 patches of the pre-processed original training images and the corresponding ground truth
  - `test190000_Original_GroundTruth_Prediction*.png` from top to bottom, the original pre-processed image, the ground truth and the prediction. In the predicted image, each pixel shows the vessel predicted probability, no threshold is applied.



## Running the experiment on DRIVE
The code is written in Python, it is possible to replicate the experiment on the DRIVE database by following the guidelines below.


### Prerequisities
The neural network is implemented by pytorch.

The following dependencies are needed:
- numpy >= 1.11.1
- PIL >=1.1.7
- opencv >=2.4.10
- h5py >=2.6.0
- configparser >=3.5.0b2
- scikit-learn >= 0.17.1
- matplotlib
- pytorch >= 1.5

Also, you will need the DRIVE database, which can be freely downloaded as explained in the next section.

### Training

First of all, you need the DRIVE database. We are not allowed to provide the data here, but you can download the DRIVE database at the official [website](http://www.isi.uu.nl/Research/Databases/DRIVE/). Extract the images to a folder, and call it "DRIVE", for example. This folder should have the following tree:
```
DRIVE
│
└───test
|    ├───1st_manual
|    └───2nd_manual
|    └───images
|    └───mask
│
└───training
    ├───1st_manual
    └───images
    └───mask
```
We refer to the DRIVE website for the description of the data.

It is convenient to create HDF5 datasets of the ground truth, masks and images for both training and testing.
In the root folder, just run:
```
python prepare_datasets_DRIVE.py
```
The HDF5 datasets for training and testing will be created in the folder `./DRIVE_datasets_training_testing/`.  
N.B: If you gave a different name for the DRIVE folder, you need to specify it in the `prepare_datasets_DRIVE.py` file.

Now we can configure the experiment. All the settings can be specified in the file `configuration.txt`, organized in the following sections:  
**[data paths]**  
Change these paths only if you have modified the `prepare_datasets_DRIVE.py` file.  
**[experiment name]**  
Choose a name for the experiment, a folder with the same name will be created and will contain all the results and the trained neural networks.  
**[data attributes]**  
The network is trained on sub-images (patches) of the original full images, specify here the dimension of the patches.  
**[training settings]**  
Here you can specify:  
- *N_subimgs*: total number of patches randomly extracted from the original full images. This number must be a multiple of 20, since an equal number of patches is extracted in each of the 20 original training images.
- *inside_FOV*: choose if the patches must be selected only completely inside the FOV. The neural network correctly learns how to exclude the FOV border if also the patches including the mask are selected. However, a higher number of patches are required for training.
- *N_epochs*: number of training epochs.
- *batch_size*: mini batch size.


After all the parameters have been configured, you can train the neural network with:
```
python retinaNN_traning.py
```
If available, a GPU will be used.  
The following files will be saved in the folder with the same name of the experiment:
- model architecture (json)
- picture of the model structure (png)
- a copy of the configuration file
- model weights at last epoch (HDF5)
- model weights at best epoch, i.e. minimum validation loss (HDF5)
- sample of the training patches and their corresponding ground truth (png)


### Evaluate the trained model
The performance of the trained model is evaluated against the DRIVE testing dataset, consisting of 20 images (as many as in the training set).

The parameters for the testing can be tuned again in the `configuration.txt` file, specifically in the [testing settings] section, as described below:  
**[testing settings]**  
- *best_last*: choose the model for prediction on the testing dataset: best = the model with the lowest validation loss obtained during the training; last = the model at the last epoch.
- *full_images_to_test*: number of full images for testing, max 20.
- *N_group_visual*: choose how many images per row in the saved figures.
- *average_mode*: if true, the predicted vessel probability for each pixel is computed by averaging the predicted probability over multiple overlapping patches covering the same pixel.
- *stride_height*: relevant only if average_mode is True. The stride along the height for the overlapping patches, smaller stride gives higher number of patches.
- *stride_width*: same as stride_height.
- *nohup*: the standard output during the prediction is redirected and saved in a log file.

The section **[experiment name]** must be the name of the experiment you want to test, while **[data paths]** contains the paths to the testing datasets. Now the section **[training settings]** will be ignored.

Run testing by:
```
python retinaNN_predict.py
```
If available, a GPU will be used.  
The following files will be saved in the folder with same name of the experiment:
- The ROC curve  (png)
- The Precision-recall curve (png)
- Picture of all the testing pre-processed images (png)
- Picture of all the corresponding segmentation ground truth (png)
- Picture of all the corresponding segmentation predictions (png)
- One or more pictures including (top to bottom): original pre-processed image, ground truth, prediction
- Report on the performance

All the results are referred only to the pixels belonging to the FOV, selected by the masks included in the DRIVE database

