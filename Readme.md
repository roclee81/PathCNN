# Efficient pan-cancer whole-slide image classification using convolutional neural networks

Code acompaining paper: [Efficient pan-cancer whole-slide image classification and outlier detection using convolutional neural networks ](https://www.biorxiv.org/content/early/2019/05/14/633123.full.pdf)

# Data:

* To demo the software/code described in the manuscript you can download the [GDC data transfer API](https://gdc.cancer.gov/access-data/gdc-data-transfer-tool)
* Create a manifest by selecting Cases > CANCER_TYPE and Files > Data Type > Tissue Slide Image.
* Download the manifest into ```manifest_file```
* Run the command ```gdc-client download -m manifest_file``` in Terminal

## 1. System requirements 
* software dependencies python3, PyTorch (software has been tested on Unix machine)

## 2. Installation guide
* Instructions
Clone this repo to your local machine using:
```
 git clone https://github.com/sedab/PathCNN.git
 
```
* Typical install time on a "normal" desktop computer is around 3 minutes.

## 3. Demo
* Expected train and test time are described in the manuscript. 

### 3.1. Data processing:

Note that data tiling and sorting scripts come from [Nicolas Coudray](https://github.com/ncoudray/DeepPATH/). Please refer to the README within `DeepPATH_code` for the full range of options. Additionally, note that these scripts may take a significant amount of computing power. We recommend submitting sections 2.1 and 2.2 to a high performance computing cluster with multiple CPUs.

#### 3.1.1. Data tiling
Run ```Tiling/0b_tileLoop_deepzoom2.py``` to tile the .svs images into .jpeg images. To replicate this particular project, select the following specifications:

```sh
python -u Tiling/0b_tileLoop_deepzoom2.py -s 512 -e 0 -j 28 -f jpeg -B 25 -o <OUT_PATH> "<INPUT_PATH>/*/*svs"
```

* `<INPUT_PATH>`: Path to the outer directory of the original svs files

* `<OUT_PATH>`: Path to which the tile files will be saved

* `-s 512`: Tile size of 512x512 pixels

* `-e 0`: Zero overlap in pixels for tiles

* `-j 28`: 28 CPU threads

* `-f jpeg`: jpeg files

* `-B 25`: 25% allowed background within a tile.

#### 3.1.2. Data sorting
To ensure that the later sections work properly, we recommend running these commands within `<ROOT_PATH>`, the directory in which your images will be stored:

```sh
mkdir <CANCER_TYPE>TilesSorted
cd <CANCER_TYPE>TilesSorted
```

* `<CANCER_TYPE>`: The dataset such as `'Lung'`, `'Breast'`, or `'Kidney'`

Next, run `Tiling/0d_SortTiles.py` to sort the tiles into train, valid and test datasets with the following specifications.

```sh
python -u <FULL_PATH>/Tiling/0d_SortTiles.py --SourceFolder="<INPUT_PATH>" --JsonFile="<JSON_FILE_PATH>" --Magnification=20 --MagDiffAllowed=0 --SortingOption=3 --PercentTest=15 --PercentValid=15 --PatientID=12 --nSplit 0
```

* `<FULL_PATH>`: The full path to the cloned repository

* `<INPUT_PATH>`: Path in which the tile files were saved, should be the same as `<OUT_PATH>` of step 2.1.

* `<JSON_FILE_PATH>`: Path to the JSON file that was downloaded with the .svs tiles

* `--Magnification=20`: Magnification at which the tiles should be considered (20x)

* `--MagDiffAllowed=0`: If the requested magnification does not exist for a given slide, take the nearest existing magnification but only if it is at +/- the amount allowed here (0)

* `--SortingOption=3`: Sort according to type of cancer (types of cancer + Solid Tissue Normal)

* `--PercentValid=15 --PercentTest=15` The percentage of data to be assigned to the validation and test set. In this case, it will result in a 70 / 15 / 15 % train-valid-test split.

* `--PatientID=12` This option makes sure that the tiles corresponding to one patient are either on the test set, valid set or train set, but not divided among these categories.

* `--nSplit=0` If nSplit > 0, it overrides the existing PercentTest and PercentTest options, splitting the data into n even categories. 

#### 3.1.3. Build tile dictionary

Run `Tiling/BuildTileDictionary.py` to build a dictionary of slides that is used to map each slide to a 2D array of tile paths and the true label. This is used in the `aggregate` function during training and evaluation.

```sh
python3 -u Tiling/BuildTileDictionary.py --data <CANCER_TYPE> --file_path <ROOT_PATH> --train_log /gpfs/scratch/bilals01/test-repo/logs/exp6_train.log
```
* --file_path `<ROOT_PATH>` points to the directory path for which the sorted tiles folder is stored in, same as in 2.2.

* --data `<CANCER_TYPE>` is the base name for the given type. 

*  train_log points to the log file from training. (This option is only needed if you are testing data where not all the classes are presented )

Note that this code assumes that the sorted tiles are stored in `<ROOT_PATH><CANCER_TYPE>TilesSorted`. If you do not follow this convention, you may need to modify this code.

### 3.2 Train model:

Run `train.py` to train with our CNN architecture. sbatch file `run_job.sh` is provided as an example script for submitting a GPU job for this script. Inside the run_job.sh, set parameteres: nexp, output and param as described below. You need to create two directories where the output of the training will be saved at: one for experiemnts, and one for logs. 

* `exp_name` = "exp8" 

* `nexp` = "dir/experiments/${exp_name}" : will create a subfolder under experiments folder which will save the checkpoints and predcitions, experiment name is determined by  the param `exp_name`

* `output` = "dir/logs/${exp_name}.log" : will create a log file under the logs folder where the training output will be printed on,log name is determined by  the param `exp_name`

* `nparam` = "--cuda  --augment --dropout=0.1 --nonlinearity='leaky' --init=‘xavier’ --root_dir=/gpfs/scratch/bilals01/brain-kidney-lung/brain-kidney-lungTilesSorted/ --num_class=7 --tile_dict_path=/gpfs/scratch/bilals01/brain-kidney-lung/brain-kidney-lung_FileMappingDict.p"

The model checkpoints at every epoch and steps (frequency determined by the user using step_freq) will be saved at experiments/checkpoints folder. And the **validation set** predictions and labels will be saved under experiments/outputs folder if calc_val_auc argument is used (Note that total training time will increase significantly if you choose to use this option).

**nparam:**
* `--cuda`: enables cuda

* `--ngpu`: number of GPUs to use (default=1)

* `--augment`: use data augmentation or not

* `--batchSize`: batch size for data loaders (default=32)

* `--imgSize`: the height / width that the image will be shrunk to (default=299)

* `--metadata`: use metadata or not

**IMPORTANT NOTE: this option is not fully implemented!** Please see section 6 for additional information about using the metadata. 

* `--nc`: input image channels + concatenated info channels if metadata = True (default = 3 for RGB).

* `--niter`: number of epochs to train for (default=25)

* `--lr`: learning rate for the optimizer (default=0.001)

* `--decay_lr`: activate decay learning rate function

* `--optimizer`: Adam, SGD or RMSprop (default=Adam)

* `--beta1`: beta1 for Adam (default=0.5)

* `--earlystop`: use early stopping

* `--init`: initialization method (default=normal, xavier, kaiming)

* `--model`: path to model to continue training from a checkpoint (default='')

* `--experiment`: where to store samples and models (default=None)

* `--nonlinearity`: nonlinearity to use (selu, prelu, leaky, default=relu)

* `--dropout`: probability of dropout in each block (default=0.5)

* `--method`: aggregation prediction method (max, default=average)

* `--num_class`: number of classes (default=2)

* `--root_dir`: path to your sorted tiles Data directory .../dataTilesSorted/ (format="<ROOT_PATH><CANCER_TYPE>TilesSorted/")

* `--tile_dict_path`: path to your Tile dictinory path (format="<ROOT_PATH><CANCER_TYPE>_FileMappingDict.p")

* `--step_freq`: how often to save the results and the checkpoints (default=100000000; won't save any steps, as this number set very high)

* `--calc_val_auc`: save validation auc calculatio at each epoch 

### 3.3 Test model:

Run ```test.py``` to evaluate a specific model on the test/validation data, ```run_test.sh``` is the associated sbatch file. Inside the run_test.sh, set parameteres: nexp, output and param as described below. 

* `exp_name` = "exp8" : set a name for the experiment

* `test_val` = "test" : choose between test and valid

* `nexp` = "dir/experiments/${exp_name}" : same exp directory set for the training, experiment name is determined by  the param `exp_name`

* `output` = "dir/logs/${test_val}.log" : name the test log with "*_test.log", log name is determined by  the param `exp_name`

* `nparam` = "--model='epoch_2.pth' --root_dir=/gpfs/data/abl/deepomics/tsirigoslab/histopathology/Tiles/LungTilesSorted/ --num_class=7 --tile_dict_path=/gpfs/data/abl/deepomics/tsirigoslab/histopathology/Tiles/Lung_FileMappingDict.p --val=${test_val}"

**nparam:**
* `--model`: Name of model to test, e.g. `epoch_10.pth`

* `--num_class`: number of classes (default=2)

* `--root_dir`: path to your sorted tiles Data directory .../dataTilesSorted/ (format="<ROOT_PATH><CANCER_TYPE>TilesSorted/")

* `--tile_dict_path`: path to your Tile dictinory path (format="<ROOT_PATH><CANCER_TYPE>_FileMappingDict.p")

* `--val`: validation vs test (default='test', or use 'valid'), use `test_val` to set the parameter

* `--train_log`: log file from training (default='')

The output data will be dumped under experiments/experiment_name folder.

* To run the test data with multiple check points, use run_multiple_test.sh script. Set the experiment, count (epoch number to start from) and step (increment of epoch number) variables in the script accordingly.

Note: If number of classes presented is less than what the model is trained for, you will need to pass the log file created by the model as input to the test script using `--train_log` parameter


### 3.4 Evaluation:

Use test_eval.ipynb to create the ROC curves and calculate the confidence intervals. To start a jupyter notebook on bigpurple, submit the run-jupyter.sbatch script and the follow the instructions on the output file.

### 3.5 TSNE Analysis:

Once the model is trained, run ```tsne.py``` to extract the last layer weights to create the TSNE plots, ```run_tsne.sh``` is the associated sbatch file.

**sbatch run_tsne.sh "--root_dir=/gpfs/scratch/bilals01/brain-kidney-lung/brain-kidney-lungTilesSorted/ --num_class=7 --tile_dict_path=/gpfs/scratch/bilals01/brain-kidney-lung/brain-kidney-lung_FileMappingDict.p --val=test" test**

* `--num_class`: number of classes (default=2)

* `--root_dir`: path to your sorted tiles Data directory .../dataTilesSorted/ (format="<ROOT_PATH><CANCER_TYPE>TilesSorted/")

* `--tile_dict_path`: path to your Tile dictinory path (format="<ROOT_PATH><CANCER_TYPE>_FileMappingDict.p")

* `--val`: validation vs test (default='test', or use 'valid')

The output data will be saved at tsne_data folder

* Use TSNE/tsne_visualize.ipynb to visualize the results (change the input file name to match with tsne.py output files  at tsne_data as needed)


## 4. Instructions for use
* Use the model checkpoints in files in checkpoints folder and follow 4. Test model instructions in the above section.

## 5. Additional resources:

### iPython Notebooks

* ```100RandomExamples.ipynb``` visualizes of 100 random examples of tiles in the datasets
* ```Final evaluation and viz.ipynb``` provides code for visualizing the output prediction of a model, and also for evaluating a model on the test set on CPU
* ```new_transforms_examples.ipynb``` visualizes a few examples of the data augmentation used for training. One can tune the data augmentation here.

