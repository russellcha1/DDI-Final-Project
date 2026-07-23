# Predicting Falcon 9 Sonic Boom Impacts at Vandenberg

## Project Overview

With the rapidly quickening launch cadence at Vandenberg Space Force Base, an important side effect must be addressed- sonic booms. They undoubtedly affect people, and the number of launches that Vandenberg operates already raises many issues. Can knowing information about launches help estimate if a rocket will produce a sonic boom over the continental United States, especially with the scaling of the number of launches by 3 fold or more.

My research question then became:

**Can launch timing, mission type, target orbit, and booster landing location predict whether at least one monitored California station records a Falcon 9 launch or landing sonic boom that is audible from the the continental United States?**

---

## Why This Question Matters

As launch cadence increases, repeated sonic booms may affect nearby communities and infrastructure. To have a model that can predict sonic boom launches would be very helpful to alert communities that will be affected by sonic booms, plan launches to avoid having sonic booms at certain times, and/or to highlight certain launches as high risk. This project is a proof of concept and does not have operationally functional forecasting.

---

## Data

### Launch Data

Launch information comes from the Space Devs Launch Library 2 API:

```text
https://ll.thespacedevs.com/2.3.0/launches/
```

The API request filters for launches associated with Vandenberg:

```text
pad__location=11
...
limit=100
```

Relevant launch fields include:

- Launch date and time
- Mission name
- Mission type
- Target orbit
- Rocket configuration
- Booster landing location

Because the only sonic boom measurements I have are from Falcon 9 vehicles, this project's scope is limited to these rockets only.

### Sonic-Boom Measurements

The sonic-boom files contain station-level measurements from monitoring locations in California.

Important fields include:

- Launch date
- Mission name
- Sonic boom monitoring station name
- Station's latitude
- Station's longitude
- Measured launch-boom pressure
- Measured landing-boom pressure
- Booster landing location

Pressure is measured in pounds per square foot, or psf.

Some columns contain the values "ND" or "ND*", which means that no launch was detected. These values were encoded in later steps.

### Final Modeling Table

After cleaning, combining, and matching the datasets, the current event-level modeling table contains:

- 62 launches
- 44 launches that produced sonic booms greater than 0.5 psf
- 18 launches that did not produce sonic booms or produced sonic booms below 0.5 psf

The classification target is:

- 1 = At least one monitored station recorded launch or landing pressure ≥ 0.5 psf
- 0 = No available monitored-station measurement reached 0.5 psf

A zero does not denote no sonic boom happened at all, but that no boom that was above 0.5 psf occurred over CONUS according to the sensor data.

---

## Repository Structure

A recommended project structure is:

```text
sonic-boom-prediction/
├─ README.md
├─ requirements.txt
├─ data
│   ├─ sonic-boom measurement files
│   └─ README.md
├─ notebooks
│   └─ Final-SonicBoom.ipynb
├─ outputs
│   ├─ vandenberg_launches_raw.csv
│   ├─ vandenberg_launches_selected.csv
│   └─ vandenberg_column_profile.csv
```

Raw sonic boom measurements should go in the data folder, and launch records and other outputs will be saved in the outputs folder.

---

## Setup

### 1. Clone the repository

```bash
git clone <repository-url>
cd Sonic-Boom-Predictor
```

### 2. Install the required packages

```bash
pip install -r requirements.txt
```

The main libraries used by the project are:

- pandas
- numpy
- requests
- tqdm
- scikit-learn
- matplotlib
- jupyter

### 3. Start Jupyter

Open the notebook in any code editor that is compatible with Jupyter Notebook and you can get started.

---

## Running the Project

### Normal Workflow

The recommended workflow is to load the launch data already saved in the outputs folder. However, if you'd like to pull the latest data, there is a limit of 15 API calls per hour if you are not authenticated. My project, at the moment, makes 10 requests so be judicious with your requests and when you want to run all, run everything below the cell that requests this data.

### Updating the Launch Data

Run the API collection section only when the launch dataset needs to be updated. The unauthenticated API has a request limit and may return an error message saying there have been too many requests. If this happens, wait for about one hour before trying again.

---

## Data Preparation

### Sonic-Boom Cleaning

The project standardizes the pressure columns by:

- Removing extra whitespace
- Converting numeric strings to numbers
- Converting `ND` and `ND*` to `0.0`
- Preserving truly missing measurements as missing values
- Standardizing station names, coordinates, dates, and sources

A station's maximum boom measurement is then aggregated from the available launch and landing measurements.

### Event-Level Aggregation

The original measurement table contains one row for each launch and monitoring station. For classification, the data are aggregated to one row per launch. The target becomes positive when at least one monitored station records a pressure of over 0.5 psf whether it be from the launch or the landing. The station-level table is kept so that future geographic or spatial modeling can be done.

### Launch Matching

Measurement dates are matched to Falcon 9 launch records from the API. Most launches match on the exact local launch date, but some require a one day tolerance because the timestamp on the API and measurement date from the sonic boom data may fall on different calendar dates.

---

## Model Features

The current classification models use information that is usually distributed before the T-0 time.

### Numeric Features

- Launch month
- Local launch hour

### Categorical Features

- Mission type
- Target orbit
- Booster landing location

---

## Modeling Approach

### Train-Test Split

The event-level data are divided into:

- 80% training data
- 20% held-out test data

The split is stratified so that the boom and no-boom proportions remain reasonably similar in both subsets. This will make each fold be more representative of the overall data and will lead to a more robust model.

### Preprocessing

Numeric features are:

- Filled using a simple imputater with a strategy of median
- Standardized to create a more fair comparison for linear regression

Categorical features are filled using the most frequent category, one-hot encoded, or configured to handle categories that were not present in a training fold. Furthermore, all preprocessing is included inside pipelines in order to prevent data leakage from the test set.

### Models

The project compares:

1. Logistic regression
2. Random forest classification

The reason I used these models is that logistic regression can provide a simple, linear baseline while random forest classifcation can also capture nonlinear relationships between the features.

### Hyperparameter Tuning

Hyperparameters are selected using:

- Five-fold stratified cross-validation
- Grid search
- Balanced accuracy as the primary model-selection score

I decided to use stratified cross-validation to keep each fold representative of the data. Balanced accuracy takes into account the accuracy of both predictions (boom or no boom) and averages them.

---

## Evaluation Metrics

Because boom events are more common than no-boom events, ordinary accuracy can be misleading.

The primary comparison metric is balanced accuracy, which is the average accuracy of predicting booms and no booms. Balanced accuracy gives equal importance to both classes.

The project also reports:

- Accuracy
- Precision
- Recall
- F1 score
- ROC-AUC
- Average precision
- Brier score

---

## Current Results

### Logistic Regression

Held-out test performance:

- Accuracy: approximately `0.46`
- Balanced accuracy: approximately `0.33`
- Boom recall: approximately `0.67`
- No-boom recall: `0.00`

The logistic regression model did not correctly identify any of the four no-boom launches in the test set.

### Random Forest

Held-out test performance:

- Accuracy: approximately `0.62`
- Balanced accuracy: approximately `0.58`
- Boom precision: approximately `0.75`
- Boom recall: approximately `0.67`
- No-boom recall: approximately `0.50`

The random forest correctly identified:

- 6 of 9 boom launches
- 2 of 4 no-boom launches

It produced:

- 3 missed boom events
- 2 false boom predictions

### Main Finding

The random forest performed significantly better than logistic regression at distinguishing between launches that produce or do not produce sonic booms, but its balanced accuracy of approximately 0.58 indicates its predictions are based on limited data. The model appears to detect some meaningful relationships, but the available launch data are not sufficient for real, operational forecasting.

---

## Interpretation

These results show that launch timing, mission type, orbit, and landing location contain *some* information about whether a monitored boom will occur, but do not have enough information about the physics of sonic booms. The model will not produce reliable predictions. One reason that the current results are not very strong is that other launch characteristics such as azimuth and trajectory plus weather conditions play a big role in determining if a sonic boom will occur. These factors contain some sort of explanation or part of the puzzle of how sound propagates in the environment.

---

## Limitations

### Small Sample Size

The model contains only 62 matched launches. The held-out test set contains only 13 launches, so one incorrect prediction changes test accuracy by approximately 7.7 percent. These results should be interpreted as only a preliminary analysis.

### Unequal Class Distribution

Boom events are more common than no-boom events. This makes ordinary accuracy less informative, leading me to use balanced accuracy. Ordinary metrics cannot be used for some benchmark testing.

### Changing Sensor Coverage

The number and location of sonic boom monitoring stations that are used differ between launches. Some launches use many stations, while others have very few. A launch with more operating stations has more opportunities for at least one station to record a boom. This means that the target reflects the actual sonic boom *and* the available stations in the monitoring network.

### Measurement Availability

Missing landing-boom values do not necessarily mean that no landing boom occurred. They may mean that a landing measurement was not available or that the landing boom happened in another location.

### Falcon 9 Only

The sonic-boom data cover Falcon 9 missions. The model should not be applied to any other launch vehicles without additional training data.

### Missing Physics-Based Predictors

The present model does not include atmospheric profiles, detailed trajectories, or booster flight geometry. Any more tuning will not fill in the information gap present in this model.

---

## Planned Regression Extension

The next stage of this project is to improve my current classification model to be able to predict whether a sonic boom will happen or not much more accurately. After this, I plan on adding a new model to my system to estimate the maximum observed pressure when a boom is expected. The regression target will be the maximum observed launch or landing pressure in psf. I plan on using ridge regression and random forest regression. Regression performance will be evaluated primarily with mean absolute error and root mean squared error because they express error values directly in psf.

This extension will be added once the classification model is improved significantly.

---

## Future Work

The most valuable next steps are:

1. Adding information about launch trajectories
2. Adding weather data for each launch
3. Expand data with more measured launches
4. Use station and launch pad coordinates to predict footprints
5. Simulate expected annual sonic booms with more annual launches
6. Incorporate this project into the Western Range Operations dashboard

A spatial model may eventually predict:

- Which cities or communities are likely to be affected
- Expected pressure at each location
- The approximate geographic footprint above a pressure threshold

---

## Practical Takeaway

The current model is a useful proof of concept, but it should not yet be used for operational launch decisions or public-warning systems. Its main contribution is that we can now tell launch characteristics alone provide limited predictive signal, and reliable sonic-boom forecasting will require atmospheric, trajectory, and consistent monitoring data. The project establishes a workflow that can incorporate additional data sources as they become available.
