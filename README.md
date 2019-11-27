# Tackling Heat and Cooling Our Cities: The Power of Urban Surface and Canopy Data to Address Climate Change, Health, Equity and Biodiversity Challenges

### Objective:
We present an automated workflow to monitor the surface reflectivity of roofs and pavements in urban areas. We built on the methods developed in [Ban-Weiss et al. 2015a](https://www.sciencedirect.com/science/article/abs/pii/S0038092X14005088/ "Ban-Weiss et al. 2015a")  & 2015b (https://www.sciencedirect.com/science/article/abs/pii/S0038092X14005283) and scale them through cloud computing and machine learning. We use official footprint data from LA city, Microsoft building footprints and OpenStreetMap/SharedStreet API to get geometries of roofs and streets. Using open-source satellite imagery from National Agriculture Imagery Program (NAIP), ground truth measurements collected through project partners, and regression machine learning we create a high-resolution map of surface reflectivity for multiple urban areas in the United States for multiple time periods. The resulting data and maps provide an estimate of the existing surface reflectivity at a building and street-segment scale which can be superimposed with current heat vulnerability, green infrastructure, urban morphology, and urban heat data. This tool serves cities in developing and evaluating urban heat island reduction strategies and promoting extensive adoption of urban heat mitigation programs. 

### Summary:
This repository contains a comprehensive set of instructions for creating and applying models that estimate surface reflectivity of roofs and streets for different urban areas in the United States using machine learning. Example results for roofs are available for Los Angeles County (https://worldresources.maps.arcgis.com/apps/webappviewer/index.html?id=abb41b4ba5504d848d4715eb2537c317) and Kansas City metro area (https://worldresources.maps.arcgis.com/apps/webappviewer/index.html?id=4fba84cf6c314c7d9937ed616d7366f4). Example results for streets are available for City of Los Angeles (https://worldresources.maps.arcgis.com/apps/webappviewer/index.html?id=9e026352d94148319272896336fb0035) 

The workflow is compressed within a set of Jupyter notebooks. The step by step instructions within the notebooks are the best way to understand the whole workflow. The training and prediction of the model is done using Azure Machine Learning Studio.

### Requirements:
All listed notebooks are written for Python 3.6. The libraries and packages required to execute the notebooks are listed in the imports block at the beginning of each. In general, these are standard geospatial and data analysis Python libraries.
Several parts of the workflow utilize the `descarteslabs` package for imagery retreival or geospatial tiling. This is the Python API provided by Descartes Labs that provides access to its "data refinery" capabilities. Utilizing the API requires registration and token generation, as described in their documentation. Unaffiliated users may not have access to all offerings, such as certain remote sensing products. 

### Inputs:
#### Training/validation: 
* measured or known albedo values of specific materials/sites at specific times. We collected roof and pavement albedo measurements from roof manufactures installers and researchers. Currently we have known albedo values associated with specific measurement or installation dates for over 30,000 unique roofs or pavement location in approximately 45 US states. However, most of the roof samples were associated with high albedo values and there were very small number of samples with low albedo values. So, we employed two techniques to cover up the lack of training data: pixel sampling and SMOTE. Rather than using all the pixels within a roof or street, we selected 20 random pixels as input. This gave us the opportunity to select multiple sample of 20 pixels from underrepresented cases. SMOTE (Synthetic Minority Oversampling Technique) is a method in Azure Machine Learning Studio (classic) to increase the number of minority cases in a dataset used for machine learning. This statistical technique helped us to create new samples with a range of albedo values for which we had no data before. Overall, 283 roof samples were used as input for the roofs model and 2294 pavement samples were used as input for the pavement model. The samples were selected in a way to have a more diverse and balanced training data.  
* geometries of training site/material. Microsoft footprint data was used as the main source of geometries for the roofs training data. For pavement, Google Direction API was used to get the centerline of the streets. The centerline was then buffered 2m in each side to get the geometry of the streets.
* high-resolution four-band imagery of training sites from within 6-months of site measurement. National Agriculture Imagery Program (NAIP) was used as the source of imagery which has 1m resolution.
#### Prediction: 
* geometries of features of interest (building footprints for roofs, street segment centerlines for streets, etc.). For roofs, official building footprints from LA city government was used for LA and Microsoft footprint data was used to acquire roof geometries for other cities. For streets, official street centerline from LA city government was used for LA and OSM street data was used for other cities. 
* high-resolution four-band imagery for features of interest for times of interest. NAIP was used as the primary source of imagery but the model was also tested on Airbus Pleiades imagery. The initial results on Airbus imagery was also very promising.

### Outputs:
* Vector dataset with estimated mean reflectivity (unitless albedo, 0-1) for each feature of interest (roof, street segment, etc.) for each time of interest. Possibility to also produce a raster version of this dataset--estimating the reflectivity at each pixel. 

### Workflow:
1.	Geocode addresses from the ground truth data for roofs
* Notebook core_geocode-ground-truth.ipynb
* Geocode the training data rooftop addresses using Google geocoding API. The latitude & longitude of the roofs will be used to search for their geometries from Microsoft footprints data.
    
    Geocode addresses from the ground truth data for streets
*	Notebook core_geocode-ground-truth_streets.ipynb
*	Geocode the training data street addresses using Google Directions API to get an encoded polyline string for the street. The polyline string is then decoded to get a list of lat/lon pairs which are then connected to get the shape of the street. 

2.	Download NAIP imagery for training data
* Notebook core_download_imagery.ipynb
* Specify the desired satellite imagery—from where, from when, including what spectral bands—and store it locally as multi-band, geospatial raster files. 
3.	Prepare training data
* Notebook core_process_training_data.ipynb
* Albedo of each feature(roof/streets) is first calculated using band values from all the pixels within a feature and utilizing an equation from Ban-Weiss et al. This calculated albedo is then used to draw the histogram for all pixels within the feature. The pixels are then grouped based on the calculated albedo values and using natural breaks. The expected albedo value for the feature is then used to find the optimum group of pixels. The algorithm searches for the group of pixels that contains the expected albedo value and then it checks whether that group of pixel contains at least 20% of total pixels. If both conditions are satisfied, that group of pixels are selected for future analysis. If not, the algorithm searches for the closest group of pixels. The search goes on until both conditions are met. From the final selection of pixels, 20 random pixels are selected for the ultimate analysis. Due to the low amount of data for low albedo roofs, multiple samples of 20 pixels are taken from roofs with low expected albedo. The band values from the selected pixels will be used as an input for the model.
4.	Normalize training data
* Notebook core_normalize-training_data.ipynb
* All the unique imagery that cover the training data are identified. The mean of each bands for all the pixels from the imagery set are evaluated and put into separate lists (four lists for the four bands). The mean and standard deviation of those lists are then calculated and used to normalize the training data. For each training data samples, their mean band values are subtracted by these calculated means and divided by these standard deviations. This final normalized set training data is used to train the model.
5.	Train the model
* The training data created from step 4 is uploaded to Azure Machine Learning studio. A model is then trained using built in Decision Forest algorithm from the studio. The model is tuned to optimize for Relative Mean Squared Error using “Tune Model Hyperparameter” tool from Azure ML studio.
The experiment used to train the model is published in Azure AI Gallery:https://gallery.cortanaintelligence.com/Experiment/Train-a-decision-forest-model-to-predict-surface-reflectivity-of-roofs. Anyone with a Microsoft account can run this experiment and get the same model.
6.	Prepare prediction data
* Notebook core_process_prediction_data.ipynb
* The two primary prediction study area are Los Angeles county and Kansas City. For LA, the official building footprint data from LA city government are used to get the geometry of the rooftops. For Kansas City, the rooftop geometries are acquired from Microsoft footprint data. The geometries are then used to acquire NAIP imagery for all the roofs within the AOI. The mean band values (R, G, B, NIR) for each rooftop are saved and used for subsequent operation. 
7.	Normalize prediction data
* Notebook core_normalize-prediction_data.ipynb
* The mean and standard deviation of all the mean band values from all the rooftop in the prediction dataset are calculated and used to normalize the prediction data. For each prediction data samples, their mean band values are subtracted by these calculated means and divided by these standard deviations.
The normalization is done separately for each study area and each time period. For example, while normalizing the 2009 prediction data for LA, the mean band values from 2009 LA imagery is only used.
8.  The model trained in step 5 is used to make prediction on the normalized mean band values associated with each roof in the prediction data. This step is done in Azure Machine Learning Studio. The experiment used to make prediction on a sample data using the trained model is also published in Azure AI Gallery: https://gallery.cortanaintelligence.com/Experiment/Predict-surface-reflectivity-for-roofs. Anyone with a Microsoft Account can copy this experiment to their own Azure ML Studio environment and make prediction on their own data using our model. For the experiment to run successfully, the column names in the user input data must match the column names in our sample data. Also, to get a reliable estimate of surface reflectivity, the user needs to normalize their prediction data using the similar technique mentioned here.

### Results:
#### Model accuracy and precision scores:
Validation score for the model: Before training, the data was split into a 70:30 ratio. 70% data was used for training and the remaining 30% was reserved for validation.
* mean absolute error from validation set: 0.010152
* root mean squared error from validation set: 0.013993
* coefficient of determination from validation set: 0.997422

Average precision scores: For buildings present in multiple images we were able to produce multiple predictions. The duplicate predictions were used to estimate the precision error of the albedo values. The scores from multiple years were averaged to get an overall precision score.
* mean absolute error from duplicate predictions: 0.0034
* root mean squared error from duplicate predictions: 0.0322

#### Comparison to Ban-Weiss prediction:
Visual comparison:

Ban-Weiss prediction for LA city 2009:
![Ban-Weiss prediction for LA city 2009:](https://github.com/wri/UrbanHeatMitigation/blob/master/Ban-Weiss%20prediction%20for%20LA%20city%202009.png)

WRI prediction for LA city 2009:
![WRI prediction for LA city 2009:](https://github.com/wri/UrbanHeatMitigation/blob/master/WRI%20prediction%20for%20LA%20city%202009.png)

#### Analytical comparison:
We computed roof by roof comparison scores to compare our prediction to the prediction from Ban-Weiss. For this analysis, we made a spatial join between the two prediction dataset. This allowed us to measure the difference in albedo prediction for each roof. The scores are:  
* Mean absolute error: 0.039
* Root mean squared error: 0.066

We also computed the relative accuracy between our prediction and the prediction from Ban Weiss. The plot below represents the order of predictions, with Ban Weiss prediction in the x-axis and our prediction in the y-axis.

![RelativeAccuracy](https://github.com/wri/UrbanHeatMitigation/blob/master/RelativeAccuracy_BanWeiss-WRI.png)

The figure suggests that overall there is a similar trend between the two predictions. However, some of the roofs with low albedo prediction from Ban Weiss changes to high albedo in our result which is also evident from the two histograms.

#### Year to Year comparison:
The change in albedo between multiple years is an important factor to check the validity of the predictions. Assuming that average albedo of a city doesn’t change significantly over a 2-5 year period, we expect a consistent predictions from year to year. So, we measured the difference in albedo prediction for each roof in multiple time periods to track it’s change of over time. Here is an example histogram displaying change in albedo between 2009 and 2012: 

![histogram_change_2009-12](https://github.com/wri/UrbanHeatMitigation/blob/master/histogram_change_2009-12.png)

The mean change looks to be approximately 0. One standard deviation is about 0.09. Nearly 75 thousand roofs had no essentially no change over the 3-year time period. 
We also wanted to get an idea about how roofs with certain albedo values are changing over time. So, we created a scatterplot to see how roofs in each albedo range changed between 2009 and 2012. 

![scatter_change_2009-12](https://github.com/wri/UrbanHeatMitigation/blob/master/scatter_change_2009-12.png)

The scatterplot shows that the roofs with low albedo predictions (between 0 and 0.3) were more consistent over time. There was a decrease in predicted albedo from 2009 to 2012 for high albedo roofs.

### Sample final output of albedo map
![Sample output](https://github.com/wri/UrbanHeatMitigation/blob/master/sample_output.PNG)

### Limitations:
* Lack of training data especially for roofs with low measured albedo.
* One of the main challenges in this project was to find a correct method to normalize the imagery from year to year. Our current normalization process has produced much improved result but there are still areas of improvement. 

### Future Work:
* Predictions for additional geographies
* Collect more training data
* Refine scene normalization process


### Acknowledgements
Thanks to support from a 2019 Microsoft AI for Earth grant, Global Cool Cities Alliance (GCCA), City of Los Angeles, George Ban-Weiss from University of Southern California, Sika AG, Federal Highway Administration Albedo Study, James E Alleman and Michael Heitzman.
