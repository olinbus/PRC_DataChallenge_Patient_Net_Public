# PRC_DataChallenge_Patient_Net_Public
PRC Data Challenge public submission repository for team_patient_net

# Additional databases / datasets used
## Kerosene-Type Jet Fuel Prices: U.S. Gulf Coast (WJFUELUSGULF)
Source: https://fred.stlouisfed.org/series/WJFUELUSGULF  
Idea being that maybe the fuel price impacts flight planning or ticket pricing and with this fuel onboard and pax loading.

## FAA Aircraft Characteristics Data
Source: https://www.faa.gov/airports/engineering/aircraft_char_database/data  
Idea being to add some global aircraft characteristics for the machine learning model to consider.  

These datasets are explored and prepared here: https://github.com/olinbus/PRC_DataChallenge_Patient_Net_Public/blob/main/notebooks/Step%202%20-%20Other%20databases%20-%20Fuel%20%26%20FAA.ipynb

*Outlook*: The request was made to get access to the BADA data, but not granted (yet).

# Parquet files processing
To overcome network as well as disc space limitations, the *.parquet files are downloaded and processed in one step. Processing is based on the traffic library in order to use the default filter, resample the data to 10s and only keep the first 20min (to cover the climb).  So, basically using:
```python
        # Load *.parquet file into traffic object
        t = Traffic.from_file(strLocFullFile)
        # clean = filter + only first 20min + resample to '10s'
        t_clean = t.first(minutes=20).filter().resample('10s').eval()
```

See: https://github.com/olinbus/PRC_DataChallenge_Patient_Net_Public/blob/main/notebooks/Step%209%20-%20Download%20%26%20Clean%20PRC%20-%20cleaned.ipynb

*Outlook*: More efforts would be need to better clean the data.

# Climb feature extraction
Using the *.parquet flight tracks, features characterising the climb in order to predict the TOW are computed. Key elements are:
- Converting the ground speed and wind data into true airspeed (TAS)
- Computing rate of climb (RoC) in the segment FL100 to FL200 (as a compromise considering data availability and quality) and later also FL50 to FL100.
- Computing "specific energy rate" using change of TAS and ROC. This being inspired by https://enac.hal.science/hal-01002401/document and https://ntrs.nasa.gov/api/citations/20230014062/downloads/TMPI_DASC23_AidaRohani.pdf

See: https://github.com/olinbus/PRC_DataChallenge_Patient_Net_Public/blob/main/notebooks/Step%2011%20-%20Compute%20Track%20Features%20-%20Simple%20RoC%20and%20TAS.ipynb

*Outlook*: More efforts would be needed to explore for which climb segments the most tracks of sufficient quality are available. Furthermore, a number of simplifications were made (e.g. neglecting the temperature effect on RoC, failed attempt to extract a good feature to account for the effect track changes / turns have on the climb specific energy rate).

# Descent feature extraction
Analogous to the above described climb features, features for the descent are computed. Key idea was to use these in particular for those flights for which no or no good quality of climb feature are available.

See: https://github.com/olinbus/PRC_DataChallenge_Patient_Net_Public/blob/main/notebooks/Step%2011%20-%20Compute%20Track%20Features%20-%20Simple%20RoD.ipynb

*Outlook*: Baseline for these descent features are not the final *.parquet files, but the outdated version. Reprocessing would be advisable. Also only FL300 down to FL200 was considered as a first shot. No checking and cleaning applied yet.

# TOW prediction models
The actual TOW prediction is based on two XGBoost models. 

See: https://github.com/olinbus/PRC_DataChallenge_Patient_Net_Public/blob/main/notebooks/Step%2013%20-%20Second%20Submisson%20Model%20-%20cleaned.ipynb

## Global model on all flights¶
The first XGBoost model called "global" in the code is intended to capture the "statistics" and "patterns" given in the challenge_set.csv. 

Key features of the model input data preparation include
- Considering the fuel price dataset.
- Considering MTOW and MLW instead of aircraft_type.
- Computing and using further date of flight features.
- Performing and using a one-hot encoding on some of the categorical data.
- Computing and using the median of the flight_distance.

This model is applied to all flights in the submission set.

*Outlook*: More efforts would be needed to understand and reduce the overfitting of this model. Likely the model can be further improved by using different hyper paramter.

## Refined model on flights with reasonable RoC feature
The second model is called "refined" in the code and is only trained, tested and applied on flights with reasonable RoC data.

Key features of the model input data preparation include
- Using the predicted TOW from the "global" model divided by the respective MTOW.
- Using different (and too many?) features extracted from the climb phase.
- Using a few features extracted from the descent phase.

This model is applied to all flights with reasonable RoC in the submission set.

*Outlook*: More efforts would be needed to understand and reduce the overfitting of this model. Also the choice of inputs is still subject to investigations. Likely the model can be further improved by using different hyper paramter.

## Submission set prediction
The submission set actual TOW prediction uses as baseline the "global" model and for flights with reasonable RoC the "refined" model.

*Outlook*: More efforts would be needed if the "refined" model should maybe only be applied to flights where the "global" model is not already better than the refined. Also an idea being to use to the descent features only or mainly for the flights for which no or no good climb features could be extracted
