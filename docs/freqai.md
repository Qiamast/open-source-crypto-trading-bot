# Freqai

!!! Note
        Freqai is still experimental, and should be used at the user's own discretion.

Freqai is a module designed to automate a variety of tasks associated with
training a regressor to predict signals based on input features. Among the
the features includes:

* Easy large feature set construction based on simple user input
* Sweep model training and backtesting to simulate consistent model retraining through time
* Smart outlier removal of data points from prediction sets using a Dissimilarity Index.
* Data dimensionality reduction with Principal Component Analysis
* Automatic file management for storage of models to be reused during live
* Smart and safe data standardization
* Cleaning of NaNs from the data set before training and prediction.

TODO:
* live is not automated, still some architectural work to be done

## Background and vocabulary

**Features** are the quantities with which a model is trained. $X_i$ represents the
vector of all features for a single candle. In Freqai, the user
builds the features from anything they can construct in the strategy.

**Labels** are the target values with which the weights inside a model are trained
toward. Each set of features is associated with a single label, which is also
defined within the strategy by the user. These labels look forward into the
future, and are not available to the model during dryrun/live/backtesting.

**Training** refers to the process of feeding individual feature sets into the
model with associated labels with the goal of matching input feature sets to
associated labels.

**Train data** is a subset of the historic data which is fed to the model during
training to adjust weights. This data directly influences weight connections
in the model.

**Test data** is a subset of the historic data which is used to evaluate the
intermediate performance of the model during training. This data does not
directly influence nodal weights within the model.

## Configuring the bot
### Example config file
The user interface is isolated to the typical config file. A typical Freqai
config setup includes:

```json
    "freqai": {
                "timeframes" : ["5m","15m","4h"],
                "full_timerange" : "20211220-20220220",
                "train_period" : "month",
                "backtest_period" : "week",
                "identifier" :  "unique-id",
                "base_features": [
                        "rsi",
                        "mfi",
                        "roc",
                ],
                "corr_pairlist": [
                        "ETH/USD",
                        "LINK/USD",
                        "BNB/USD"
                ],
                "train_params" : {
                        "period": 24,
                        "shift": 2,
                        "drop_features": false,
                        "DI_threshold": 1,
                        "weight_factor":  0,
                },
                "SPLIT_PARAMS" : {
                    "test_size": 0.25,
                    "random_state": 42
                },
                "CLASSIFIER_PARAMS" : {
                    "n_estimators": 100,
                    "random_state": 42,
                    "learning_rate": 0.02,
                    "task_type": "CPU",
                },
        },

```

### Building the feature set

Most of these parameters are controlling the feature data set. The `base_features`
indicates the basic indicators the user wishes to include in the feature set.
The `timeframes` are the timeframes of each base_feature that the user wishes to
include in the feature set. In the present case, the user is asking for the
`5m`, `15m`, and `4h` timeframes of the `rsi`, `mfi`, `roc`, etc. to be included
in the feature set.

In addition, the user can ask for each of these features to be included from
informative pairs using the `corr_pairlist`. This means that the present feature
set will include all the `base_features` on all the `timeframes` for each of
`ETH/USD`, `LINK/USD`, and `BNB/USD`.

`shift` is another user controlled parameter which indicates the number of previous
candles to include in the present feature set. In other words, `shift: 2`, tells
Freqai to include the the past 2 candles for each of the features included
in the dataset.

In total, the number of features the present user has created is:_

no. `timeframes` * no. `base_features` * no. `corr_pairlist` * no. `shift`_
3 * 3 * 3 * 2 = 54._

### Deciding the sliding training window and backtesting duration

`full_timerange` lets the user set the full backtesting range to train and
backtest through. Meanwhile `train_period` is the sliding training window and
`backtest_period` is the sliding backtesting window. In the present example,
the user is asking Freqai to train and backtest the range of `20211220-20220220` (`month`).
The user wishes to backtest each `week` with a newly trained model. This means that
Freqai will train 8 separate models (because the full range comprises 8 weeks),
and then backtest the subsequent week associated with each of the 8 training
data set timerange months. Users can think of this as a "sliding window" which
emulates Freqai retraining itself once per week in live using the previous
month of data.


## Running Freqai
### Training and backtesting

The freqai training/backtesting module can be executed with the following command:

```bash
freqtrade backtesting --strategy FreqaiExampleStrategy --config config_freqai.example.json --freqaimodel ExamplePredictionModel
```

where the user needs to have a FreqaiExampleStrategy that fits to the requirements outlined
below. The ExamplePredictionModel is a user built class which lets users design their 
own training procedures and data analysis. 

### Building a freqai strategy

The Freqai strategy requires the user to include the following lines of code in `populate_ any _indicators()`

```python
        from freqtrade.freqai.strategy_bridge import CustomModel

        def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
                # the configuration file parameters are stored here
                self.freqai_info = self.config['freqai']

                # the model is instantiated here
                self.model = CustomModel(self.config)

                print('Populating indicators...')

                # the following loops are necessary for building the features 
                # indicated by the user in the configuration file.
                for tf in self.freqai_info['timeframes']:
                        dataframe = self.populate_any_indicators(metadata['pair'],
                                                                dataframe.copy(), tf)
                        for i in self.freqai_info['corr_pairlist']:
                        dataframe = self.populate_any_indicators(i,
                                        dataframe.copy(), tf, coin=i.split("/")[0]+'-')

                # the model will return 4 values, its prediction, an indication of whether or not the prediction 
                # should be accepted, the target mean/std values from the labels used during each training period.
                (dataframe['prediction'], dataframe['do_predict'], 
                        dataframe['target_mean'], dataframe['target_std']) = self.model.bridge.start(dataframe, metadata)

                return dataframe
```
The user should also include `populate_any_indicators()` from `templates/FreqaiExampleStrategy.py` which builds 
the feature set with a proper naming convention for the IFreqaiModel to use later.

### Building an IFreqaiModel

Freqai has a base example model in `templates/ExamplePredictionModel.py`, but users can customize and create
their own prediction models using the `IFreqaiModel` class. Users are encouraged to inherit `train()`, `predict()`, 
and `make_labels()` to let them customize various aspects of their training procedures.

### Running the model live

After the user has designed a desirable featureset, Freqai can be run in dry/live
using the typical trade command:

```bash
freqtrade trade --strategy FreqaiExampleStrategy --config config_freqai.example.json --training_timerange '20211220-20220120'
```

Where the user has now specified exactly which of the models from the sliding window
that they wish to run live using `--training_timerange` (typically this would be the most
recent model trained). As of right now, freqai will
not automatically retain itself, so the user needs to manually retrain and then
reload the config file with a new `--training_timerange` in order to update the
model.


## Data anylsis techniques
### Controlling the model learning process

The user can define model settings for the data split `data_split_parameters` and learning parameters
`model_training_parameters`. Users are encouraged to visit the Catboost documentation
for more information on how to select these values. `n_estimators` increases the
computational effort and the fit to the training data. If a user has a GPU
installed in their system, they may benefit from changing `task_type` to `GPU`.
The `weight_factor` allows the user to weight more recent data more strongly
than past data via an exponential function:

$$ W_i = \exp(\frac{-i}{\alpha*n}) $$

where $W_i$ is the weight of data point $i$ in a total set of $n$ data points._

`drop_features` tells Freqai to train the model on the user defined features,
followed by a feature importance evaluation where it drops the top and bottom
performing features (there is evidence to suggest the top features may not be
helpful in equity/crypto trading since the ultimate objective is to predict low
frequency patterns, source: numerai)._

Finally, `period` defines the offset used for the `labels`. In the present example,
the user is asking for `labels` that are 24 candles in the future.

### Removing outliers with the Dissimilarity Index

The Dissimilarity Index (DI) aims to quantiy the uncertainty associated with each
prediction by the model. To do so, Freqai measures the distance between each training
data point and all other training data points:

$$ d_{ab} = \sqrt{\sum_{j=1}^p(X_{a,j}-X_{b,j})^2} $$

where $d_{ab}$ is the distance between the standardized points $a$ and $b$. $p$
is the number of features i.e. the length of the vector $X$. The
characteristic distance, $\overline{d}$ for a set of training data points is simply the mean
of the average distances:

$$ \overline{d} = \sum_{a=1}^n(\sum_{b=1}^n(d_{ab}/n)/n) $$

$\overline{d}$ quantifies the spread of the training data, which is compared to
the distance between the new prediction feature vectors, $X_k$ and all the training
data:

$$ d_k = \argmin_i d_{k,i} $$

which enables the estimation of a Dissimilarity Index:

$$ DI_k = d_k/\overline{d} $$

Equity and crypto markets suffer from a high level of non-patterned noise in the
form of outlier data points. The dissimilarity index allows predictions which
are outliers and not existent in the model feature space, to be thrown out due
to low levels of certainty. The user can tweak the DI with `DI_threshold` to increase
or decrease the extrapolation of the trained model.

### Reducing data dimensionality with Principal Component Analysis

TO BE WRITTEN

## Additional information
### Feature standardization

The feature set created by the user is automatically standardized to the training
data only. This includes all test data and unseen prediction data (dry/live/backtest).

### File structure

`user_data_dir/models/` contains all the data associated with the trainings and
backtestings. This file structure is heavily controlled and read by the `DataHandler()`
and should thus not be modified. 