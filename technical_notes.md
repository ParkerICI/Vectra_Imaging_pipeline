# Documentation for Vectra Analysis Pipeline

Multiplexed imaging data is providing dramatic opportunities to understand complex tissues, but there is an acute need for better analysis tools. For example, cancer progression and metastasis involve a complex interplay between heterogeneous cancer cells and the tumor microenvironment (TME). Recent research into cancer therapeutic options, including immune-checkpoint inhibitors and novel targeted agents, has emphasized the prognostic and pathophysiological roles of the TME. Multiplexed imaging can provide deep characterization of the TME and other tissues of interest. Here, we provide a pipeline for multiplexed imaging quality control and processing. It contains three core steps: 

1. Preprocess raw images to remove undesired noise (introduced by technical sources) while retaining biological signal.  
2. Perform segmentation to draw boundaries around individual cells, making it possible to discern morphology and to determine which features, such as detected RNA or protein, belong to each cell.  
3. Extract cellular features from images via segmentation and assign cell types to each cell. 

To transform digital images into cell-level measurements, we have been applying, comparing and optimizing cutting-edge computer vision and machine learning techniques to each of the steps above. All code is written in Python.



## Installation 

* Clone the GitHub [repository](https://github.com/dpeerlab/)

* Create a new conda environment for this pipeline 

* ```bash
  conda env create -n davinci -f environment.yml 
  source activate davinci
  python -m ipykernel install --user --name davinci
  ```

* Install this package

* ```bash
  python setup.py install
  ```

* Install the deep learning model for segmentation following the instruction on [link](https://github.com/dpeerlab/Mask_R-CNN_cell)



## Usage

### 0. Data inspection

The first step in the analysis is to inspect the experimental setup and examine the data, by confirming the data composition and signal distribution, and visualizing the images in RGB (Red-Green-Blue) space.

#### 0.1. Image file inspection [Notebook](./notebook/0.1.Image_file_inspection.ipynb)

First, check the composition of the image files, including information such as sample ID, image size and markers present. Then remove extraneous samples, check for missing samples and remove duplicated images.

Step 1: Input the path of the raw TIF data and use the `glob` function to find all the pathnames matching a specified pattern according to the rules used by the Unix shell. Example:

```python
folder_path = '../data/tif/*.tif'
file_list = sorted(glob.glob(folder_path))
```

Step 2: Open the image files in a loop and record their size information. The `np.unique` function can be used to determine the number of images of each size. This information, together with magnification, will help to ascertain the physical size of the image. One cohort will sometimes include images with different sizes, which has important implications for downstream analysis.

Step 3: Parse marker information from the file metadata. If the information is stored during data generation, we can parse it out from metadata in the TIF file using `tifffile`. Otherwise, enter marker information manually. It is very important to make sure that the marker information is correct; we recommend double-checking the actual image for each marker to confirm this.

Step 4: Check for extra, redundant or missing samples in the cohort dataset. To remedy this, first define a cohort ID list, either manually or using another dataset. Then parse the sample ID from the image filename to identify extraneous and missing samples. Finally, remove redundant data using `shutil.move`, and contact the data provider about missing data.

Step 5. Check data composition again after the correction.

#### 0.2. Image data inspection [Notebook](./notebook/0.2.Image_data_inspection.ipynb)

In an experimental design with multiple samples, data normalization is very important. This step of the pipeline examines the signal intensity distribution and finds optimized normalization parameters. Maximal signal intensity is a key parameter in fluorescence imaging.

Step 1. Obtain the image file list using `glob`, load the image with `tifffile.imread` and smooth the data with `ndimage.gaussian_filter` from the scipy package to remove outliers. 

Step 2. Display the distribution of max intensity from each image and sample to have an overall impression of each channel. This step indicates how the max intensity looks in marker-positive and marker-negative images, and how max values differ from each other in positive images.

Step 3. Normalize signal intensities. Normalization is based on the maximum signal intensity for each image, which is determined differently depending on whether a marker is truly expressed or not in the image. It is important to note that all images are not necessarily positive for all markers; thus a threshold for calling a marker positive should first be selected. 

For marker-positive images, the max signal intensity of that image is used to normalize its intensity values.

If marker-negative images are normalized using the max signal intensity, the noise would be amplified and appear as true signal. Thus, for a given marker, we recommend first plotting the max signal intensity distribution across all samples, and identifying potential bimodality (showing max noise and max true signal); for marker-negative images, the _minimum_ of the max true signal in the global distribution usually gives the best normalization.

It is critical to visualize a selection of images with weakest and strongest signal both before and after normalization in order to fine-tune the choice of threshold. Noise should be minimized upon normalization, but weak true signals should not be lost.

#### 0.3. Image visualization  [Notebook](./notebook/0.3.Image_visualization.ipynb)

Raw images are usually in a format that contains multiple single-channel images that cannot be viewed simultaneously. To better visualize all channels in one image, some of the markers need to be converted into RGB/multiple color space.

Step 1. Get the list of target `tif` images and design a color panel for the marker. In both theory and practice, 7 colors are the limit for good visualization; in particular, `green, white (gray), cyan, yellow, magenta, blue, red`. The channel order for the color can be assigned in the function `convert_one_image` with parameter `color_list`. 

The `tqdm` function shows the progress of converting the images. When complete, converted images can be visualized in a notebook, or simply opened in windows. The images are saved in `png` format.



### 1. Preprocessing 

The preprocessing step removes undesired noise from the data. Visualization before and after preprocessing serves as a quality control check.

#### 1.1. Image preprocessing  [Notebook](./notebook/1.1.Image_preprocessing.ipynb)

Thresholding is used to remove noise and can be performed automatically, though automated thresholding is not always successful. Setting a lower boundary (to remove obvious noise) and an upper boundary (to protect against removing obvious signal) for the thresholds per sample is a way to avoid clearly unreasonable values generated by automated methods. Visualizing the images before and after thresholding can help guide fine tuning of the boundaries. The two boundaries should be recorded in a spreadsheet.

Step 1. Create blank DataFrame to store the min and max threshold information per sample and per marker. 

Step 2. Go through each channel and each sample, and set the best min and max values. In our experience (MSK Vectra data on TME samples), `min=2` and `max=4` works best; however, images with high noise or very weak true signal need to be adjusted accordingly. 

Step 3. If automatic thresholding provides ridiculous values, check the source code in the `threshold_one` and `threshold_one_plot` functions in the Vectra Tools package. 

#### 1.2. Image QC report  [Notebook](./notebook/1.2.Image_QC_report.ipynb)

Raw image data cannot usually be parsed manually -- even after converting to RGB format, there are too many images to assess individually. The function of the QC report is to group all channel images from one sample into a single file, and to enable visualization before and after preprocessing. Based on the report, preprocessing parameters can be fine-tuned, or poor quality images can be deleted from the dataset.

Step 1. Load raw image, choose a report folder and project name. 

Step 2. Generate a report for raw data. Since the sample ID and other information will be parsed from the file name, some modification of the `one_report` function might be needed to accommodate different naming schemes.

Step 3. Generate a report for images with and without preprocessing. Load the `threshold` and `max_cap` parameters from previous notebooks, and otherwise proceed in the same manner as the step above.

Step 4. Run through all the images and save the thresholds in a local file.

### 2. Segmentation 

To obtain single-cell information, the deep learning method Mask R-CNN can be used to perform instance segmentation on the RGB images. Before using Mask R-CNN, we recommend learning about the method and its installation via [link](https://github.com/dpeerlab/Mask_R-CNN_cell). GPU hardware is highly advised for training. 

#### 2.1. Train deep learning model on custom data [Notebook](./notebook/2.1.Image_segmentation_train.ipynb)

With target images and some human-annotated masks, we can train the deep learning model to predict segmentation results. We recommend 10+ training images larger than 400x400 pixels under 200X total magnification. 

Step 1. Load the training images with annotations. Each mask is assumed to be a single-channel image in which objects are disconnected. Five or more masks should be visualized to check if the images and annotations match each other. By default, the images will be randomly cropped to 128*128 pixels during training. This size provides the best results during testing, but can be changed in the `CellConfig()` function by adjusting the parameter `RPN_ANCHOR_SCALES`, which denotes the target size.

Step 2. Configure the training parameters using `init_with = "imagnet"` or `init_with = "coco"`. This transfer learning step will largely improve the results, since the pre-trained model will learn basic segmentation on natural images. Data augmentation is also applied to the training data, as shown below:

```python
augmentation = iaa.SomeOf((0, 5), [
        iaa.Fliplr(0.5),
        iaa.Flipud(0.5),
        iaa.OneOf([iaa.Affine(rotate=90),
                   iaa.Affine(rotate=180),
                   iaa.Affine(rotate=270)]),
        iaa.Affine(rotate=(-90,90)),
    iaa.Sometimes(0.2, iaa.GaussianBlur(sigma=(0.0,5.0)))
    ,iaa.Sometimes(0.2, iaa.Affine(scale={"x": (0.7, 1.5), "y": (0.7, 1.5)})),
    iaa.Sometimes(0.2, iaa.Multiply((0.7, 1.5))),
    iaa.Sometimes(0.1, iaa.Affine(shear=(-45, 45)))
])
```

Step 3. Training can begin with 10 epochs, 1000 steps per epoch, on the head layers of the model. This can be followed by 20 epochs executed on all layers, and finally, another 10 epoch on all layers using a lower learning rate. It usually takes under 1 day to complete training. Note that number of steps per epoch can be adjusted based on training data volume using `CellConfig()`.

#### 2.2. Predict segmentation with pre-trained model [Notebook](./notebook/2.2.Image_Segmentation_prediction.ipynb)

A pre-trained Mask R-CNN model can be used for prediction on images generated either with the training panel or with a different panel, so long as the new set of images share similarities with the trainig set. For example, if cells and nuclei are of similar size and shape, and tumor and immune cells have the same coloring across both image sets, then the prediction should work.

CPU hardware is recommended for this step, because it is cheaper and provides more memory.

Step 1. Load the path of the pre-trained weights.

Step 2. Set the inference parameters. `DETECTION_MAX_INSTANCES` and `POST_NMS_ROIS_INFERENCE` decide the max number of objects to be predicted in the image. The number of objects depends on image size, and is ideally chosen to represent the smallest number that still covers all objects in the image (smaller parameters allow faster prediction). `DETECTION_MIN_CONFIDENCE` is the threshold for the confidence of the prediction. `DETECTION_NMS_THRESHOLD` controls the level of crowding in the prediction. 

Step 3. Perform prediction on training images to see if your training works.

Step 4. Perform prediction on some validation images.

Step 5. Perform prediction on the list of all images. Some images can be too large to be predicted at once, so a unit prediction size such as 512*704 can be set, and stitching can then be performed on the results. 



### 3. Cell typing

From cleaned images containing normalized signal and segmented cells, it is possible to obtain spatial, morphological and expression information for each cell. This information is used to assign cell types.

#### 3.1. Cell feature extraction [Notebook](./notebook/3.1.Cell_feature_extraction.ipynb)

Step 1. Load all TIF and segmentation files. Parse their unique IDs and check if the order of files matches. 

Step 2. Load thresholding and cap dataset for preprocessing and normalization. 

Step 3. Run feature extraction. Use the `parallel` function to utilize multiple CPUs for this task. 

Step 4. Optional: change the data types as follows to optimize storage space

```
id_image             5145 non-null int16
id_sample            5145 non-null int16
id_cell              5145 non-null int32
area                 5145 non-null int16
centroid_x           5145 non-null float32
centroid_y           5145 non-null float32
orientation          5145 non-null float16
eccentricity         5145 non-null float16
minor_axis_length    5145 non-null float16
major_axis_length    5145 non-null float16
chan1_sum            5145 non-null float16
chan1_area           5145 non-null int16
chan2_sum            5145 non-null float16
chan2_area           5145 non-null int16
chan3_sum            5145 non-null float16
chan3_area           5145 non-null int16
chan4_sum            5145 non-null float16
chan4_area           5145 non-null int16
chan5_sum            5145 non-null float16
chan5_area           5145 non-null int16
chan6_sum            5145 non-null float16
chan6_area           5145 non-null int16
chan7_sum            5145 non-null float16
chan7_area           5145 non-null int16
```

#### 3.2. Cell typing [Notebook](./notebook/3.2.Cell_typing.ipynb)

This step inspects the distribution of cellular features, performs cell typing, and then visualizes the cells in each cluster to tune cell typing parameters.

Step 1. Load the feature DataFrame. 

Step 2. Display the distribution of the sum, mean and area of the signals. 

Step 3. Based on the distribution, find a cutoff for the average signal and the signal area. Next, in cases of marker co-expression in the same cells, use biological knowledge to identify cells where co-expression is not expected and remove all but the dominant, expected marker. This should be performed on a case-by-case basis. Some exceptions exist; for example, strong T-cell expression signals within a tumor region, even if they are not dominant, are likely to represent T cells, so they should be assigned as T cells and tumor expression within the same cells should be removed.

Step 4. Visualize several positive examples of marker expression for each channel and re-tune the cell typing parameters based on the visualization results.
