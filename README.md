# Introduction
We introduce a large-scale dataset called **TabFact**(website: https://tabfact.github.io/), which consists of 117,854 manually annotated statements with regard to 16,573 Wikipedia tables, their relations are classified as *ENTAILED* and *REFUTED*. The full paper is "[TabFact: A Large-scale Dataset for Table-based Fact Verification
](https://arxiv.org/pdf/1909.02164.pdf)". In this project, we aim to test the existing machine learning model's capability to handle the cases where both semantic inference and symbolic inference are involved.

<p align="center">
<img src="resource/example.png" width="700">
</p>
The table-based fact verification is the first dataset to perform fact verification on strctured data, which involves mixed reasoning in both symbolic and linguistic form. Therefore, we propose two models, namely Table-BERT and the Latent Program Algorithm to tackle this task.

- The brief architecture of Latent Program Algorithm (LPA) looks like following:
<p align="center">
<img src="resource/program.png" width="700">
</p>

- The brief architecture of Table-BERT looks like following:
<p align="center">
<img src="resource/BERT.jpg" width="700">
</p>

## News
Our Challenge is online in [CodaLab](https://competitions.codalab.org/competitions/21611), please consider submitting your system prediction to the challenge. The blind test input is in the challenge folder, it contains roughly 9.6K statements verified against the seen tables during training. Your submission format should be in:
```
  {test_id: label}, setting label=1 when it's entailed, label=0 when it's refuted.
```

## Bug Fixing Log:
1. September 11: Clean the code run with Python 3.5.
2. September 11: Added the freq_list.json to data/ folder, which is used for entity linking step.
3. September 11: Remove the old README.md under data/ folder, which is not up-to-date, the current entity linking adopts #row_num, column_num# format.

## Explore the data
We design an interface for you to browse and eplore the dataset in https://tabfact.github.io/explore.html

## Requirements
- Python 3.5
- Pytorch 1.0
- Ujson 1.35
- Pytorch 1.0+
- Pytorch_Pretrained_Bert 0.6.2 (Huggingface Implementation)
- Pandas
- tqdm-4.35
- TensorboardX
- unidecode
- nltk: wordnet, averaged_perceptron_tagger

## Direct Running: Without Preprocessing Data
### Download the synthesized program candidates
Downloading the preprocessed data for LPA
Here we provide the data we obtained after preprocessing through the above pipeline, you can download that by running

```
  sh get_data.sh
```

### Latent Program Algorithm
1. Training the ranking model
Once we have all the training and evaluating data in folder "preprocessed_data_program", we can simply run the following command to evaluate the fact verification accuracy as follows:

```
  cd code/
  python model.py --do_train --do_val
```
2. Evaluating the ranking model
We have put our pre-trained model in code/checkpoints/, the model can reproduce the exact number reported in the paper:
```
  cd code/
  python model.py --do_test --resume
  python model.py --do_simple --resume
  python model.py --do_complex --resume
```
### Table-BERT
1. Training the verification model
```
  cd code/
  python run_BERT.py --do_train --do_eval --scan horizontal --fact [first/second]
```
2. Evaluating the verification model
```
  cd code/
  python run_BERT.py --do_eval --scan horizontal --fact [first/second] --load_dir YOUR_TRAINED_MODEL --eval_batch_size N
```
### Checkpoints
1. We already put the checkpoints of LPA model under code/checkpoints, the results should be reproduced using these model files.
2. We provide the checkpoints of Table-BERT in Amazon S3 server, you can directly download it using:
```
  wget https://tablefact.s3-us-west-2.amazonaws.com/snapshot.zip
```

## Start from scratch: data preprocessing
The folder "collected_data" contains the raw data collected directly from Mechnical Turker, all the text are lower-cased, containing foreign characters in some tables. There are two files, the r1 file is collected in the first round (simple channel), which contains sentences involving less reasoning. The r2 file is collected in the second round (complex channel), which involves more complex multi-hop reasoning. These two files in total contains roughly 110K statements, the positive and negative satements are balanced. We demonstrate the data format as follows:
  ```
  Table-id: {
  [
  Statement 1,
  Statement 2,
  ...
  ],
  [
  Label 1,
  Label 2,
  ...
  ],
  Table Caption
  }
  ```
The folder "data" contains all the tables as csv files and the data splits, train_id.json indicates all the tables used for training, val_id.json indicates all the tables used for validation and test_id.json indicates testing split.


1. General Tokenization and Entity Matching
    - tokenized_data: This folder contains the data after tokenization with preprocess_data.py by:
      ```
      cd code/
      python preprocess_data.py
      ```
      this script is mainly used for feature-based entity linking, the entities in the statements are linked to the longest text span in the table cell. The result file is tokenized_data/full_cleaned.json, which has a data format like:
      ```
      Table-id: {
      [
      Statement 1: xxxxx #xxx;idx1,idx2# xxx.
      Statement 2: xx xxx #xxx;idx1,idx2# xxx.
      ...
      ],
      [
      Label 1,
      Label 2,
      ...
      ],
      Table Caption
      }
      ```
      The enclosed snippet #xxx;idx1,idx2# denotes that the word "xxx" is linked to the entity residing in idx1-th row and idx2-th column of table "Table-id.csv", if idx1=-1, it links to the table caption. The entity linking step is essential for performing  the following program search algorithm.

2. Tokenization For Latent Program Algorithm
    - preprocessed_data_program: This folder contains the preprocessed.json, which is obtained by:
      ```
      cd code/
      python run.py
      ```
      this script is mainly used to perform cache (string, number) initialization, the result file looks like:
      ```
      [
        [
        Table-id,
        Statement: xxx #xxx;idx1,idx2# (after entity linking),
        Pos-Tagging information,
        Statement with place-holder,
        [linked string entity],
        [linked number entity],
        [linked string header],
        [linked number header],
        Statement-id,
        Label
        ],
      ]
      ```
      This file is directly fed into run.py to search for program candidates using dynamic programming, which also contains the tsv files neccessary for the program ranking algorithm.

3. Latent Program Search Algorithm

    - We use the proposed latent program search algorithm in the paper to synthesize the potential candididate programs which consist with the semantics of the natural language statement. This step is based on the previous generated "preprocessed.json". The search procedure is parallelized on CPU cores, which might take a long time. Using a stronger machine with more than 48 cores is preferred. In our experiments, we use 64-core machine to search for 6 hours to obtain the results. For convience, we already add the results in "preprocessed_data_program/all_programs.json", which can be downloaded using get_data.sh script. To start the latent program synthesis, you can simply type in the following command:
    ```
      python run.py --synthesize
    ```
    
    - We will save the searched intermediate results for different statements in the temporary folder "all_programs", we save the results in different files for different statements, the format of intermediate program results look like:
      ```
      [
        csv_file,
        statement,
        placeholder-text,
        label,
        [
          program1,
          program2,
          ...
        ]
      ]
      ```
   - Finally, we gather all the intermeidate searched results and combine them into one files in "preprocessed_data_program" folder, you can perform this operation by the following command. This script will save all the neccessary train/val/test/complex/simple/small splits into "preprocessed_data_program" for the ranking model to proceed.
   ```
     python generate_ranking_data.py 
   ```
   
2. Tokenization for Table-BERT
```
  cd code/
  python preprocess_BERT.py --scan horizontal
  python preprocess_BERT.py --scan vertical
```

## Build Your Own Model
If you do not want to follow the provided data preprocessing pipeline, you can directly use the provided train_examples.json, val_examples.json and test_examples.json under tokenized_data/ folder. The text are normalized without performing entity linking.

## Instability in Table-BERT model
Throughout our experiments on Table-BERT, we found that the performance of Table-BERT will drop to random if we train too long enough, the accuracy does not converge to a reasonable range. We instead only save the best model which achieves the best performance on the validation set.
<p align="center">
<img src="resource/trend.jpg" width="550">
</p>


## Reference
Please cite the paper in the following format if you use this dataset during your research.
```
@inproceedings{2019TabFactA,
  title={TabFact : A Large-scale Dataset for Table-based Fact Verification},
  author={Wenhu Chen, Hongmin Wang, Jianshu Chen, Yunkai Zhang, Hong Wang, Shiyang Li, Xiyou Zhou and William Yang Wang},
  year={2019}
}
```

## Q&A
If you encounter any problem, please either directly contact the first author or leave an issue in the github repo.

