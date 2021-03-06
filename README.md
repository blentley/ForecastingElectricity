## Project Introduction

The purpose of this repository is to share with you my initial attempt at predicting electricity demand using a LSTM neural network for the NSW National Electricity Market (NEM) region. After reading about the successes other applications had achieved using LSTMs for sequence type predictions, I was keen to give them a try on this time series problem.  
  
This readme will give a flavour for the process I have undertaken to model this demand as it's a simplification of the detail in the full version. The full suite of scripts are also available in this repository and I will make reference to them in the text below.
  
### Project Motivations
*Why did I choose to work on forecasting in the Australian electricity market?*  
+ The energy sector has been undergoing some major transformation in recent times. The private sector and consumers are becoming increasingly anxious about our energy future as energy prices continue to rise above inflation trends, while politicians have languished in their policy positions. The result is that there seems to be something major happening in the news related to the energy market every few days. 
+ I'm passionate about renewable energy, climate change and the environment, so I wanted to understand more about how the electricity market operates.  
+ When I think about career pathways, I could see myself working in this industry. There's loads of data, technology is changing rapidly and there's often a geospatial angle to it (something I like).  
+ I had an appetiser into time-series modelling in my last semester at uni, which followed on with some forecasting at work. I wanted to extend my learning a bit further this time and see what python and LSTMs were all about.  
  
### The National Electricity Market  

The Australian Energy Market Operator (AEMO) oversees the NEM. Their [website](https://www.aemo.com.au/Electricity/National-Electricity-Market-NEM) has some excellent resources for understanding how the market operates if you would like further information. One of AEMO's functions is to make a number of short and long term forecasts for effective business planning and investment decisions. This project will focus forecasting over a short time horizon.  

### Other important technical details
The main tools I have used for this analysis are:  
 + Python 3.6, including the following libraries  
 	+ pandas  
 	+ plotly  
 	+ keras with tensorflow as the backend
 + PostgreSQL 9.6.1  
  
This analysis will focus on the NSW NEM region and air temperature at Bankstown Airport (chosen for its somewhat geographical 'average' of Sydney, NSW) for the 2016 calendar year.

### Acknowledgements  
I couldn't have made it this far without the knowledge shared by Dr Jason Brownlee and Jakob Aungiers, whose blogs were thoughtfully curated and easy to read. I'd highly encourage you to check them out. Respective links below:  
https://machinelearningmastery.com/time-series-prediction-lstm-recurrent-neural-networks-python-keras/  
http://www.jakob-aungiers.com/articles/a/LSTM-Neural-Network-for-Time-Series-Prediction  
  
I've prepared this markdown file using this helpful guide:  
https://guides.github.com/features/mastering-markdown/  
  
## Understanding the data  
The datasets I have used are quite straightforward. They are:  
+ Demand *(source: AEMO)*  
	+ Settlement date - *settlementdate (timestamp)* - dispatch settlement timestamp observed in 5 minute frequencies  
	+ NEM region - *regionid (character)* - identifier of NEM region  
	+ Demand - *totaldemand (numeric)* - demand for electricity measured in megawatts (MW)  
  
A sample of this data is available in SampleData/Demand.csv  

+ Air temperature *(source: Bureau of Meterology)*  
	+ Time - *index (timestamp)* - air temperature observation timestamp observed in 30 minute frequencies  
	+ Weather Station identifier - *station_id* (character)* - An ID code to identify weather stations around the country  
	+ Temperature - *air_temp (numeric)* - Air temperature measured in degrees Celsius  
  
A sample of this data is available in SampleData/Temperature.csv  

It's worth noting that the air temperature data will not arrive in a clean format as above. Each weather station's data will be in its own file and the usual cleansing, formatting and gap-filling will be needed.  

The scripts where I've loaded raw demand and climate data into PostgreSQL can be found in Scripts/LoadDemand.ipynb and Scripts/LoadClimate.ipynb respectively. I've also included equivalent HTML outputs.   
  
## Exploring the data  
This is what a few days of energy demand in January 2016 looked like:  
![JanuaryDemand](https://github.com/blentley/ForecastingElectricity/blob/master/Screenshots/DemandSample.PNG)  
  
What we can see from this plot is that on regular days (5th to 11th), the demand averages around the 8,000MW level, which is consistent with the NSW average. There are however, days where the peak exceeds 11,000MW. What could cause such an irregular pattern in demand?  
  
Let's add air temperature as an overlay over that same period:  
![DemandInclTemp](https://github.com/blentley/ForecastingElectricity/blob/master/Screenshots/TempOverlay.PNG)  

What is evident is that as temperature rose, so did demand.  

The BOM confirms this extreme weather event in their January 2016 summary:  
>"Two heatwaves, over 11-14 and 19-21 January, resulted Observatory Hill recording 8 days above 30 °C, well above the average of 3 days and the most hot days since January 1991."  
http://www.bom.gov.au/climate/current/month/nsw/archive/201601.sydney.shtml  

It's not a perfectly aligned because we are comparing the whole of NSW and the air temperature observed at a single location, but there's clearly a relationship present.  
    
Let's remove the time-series sequencing and plot temperature against demand to understand this relationship independent of time:  
![DemandTemp](https://github.com/blentley/ForecastingElectricity/blob/master/Screenshots/DemandTemp.PNG)  
  
We can see a clear U-shape forming between electricity demand and temperature. As temperature rises, people use more air conditioning, so demand rises. Simiarly, as temperature falls, people use more heating, so demand rises. There remains a 'sweet spot' of mild temperature between approximately 17 and 21 degrees Celsius where demand is lowest. The plot above has split the points by Weekdays and Weekends to show the comparatively lower demand on weekends due to large sections of the businesses sector not operating.  

The script where I've done my exploratory analysis can be found in Scripts/ExploratoryAnalysis.ipynb. There is also an equivalent HTML output.  

## Modelling Overview  
I started off by developing a predictive model of energy demand, where I only used historical demand as a predictor. This would set a reference point for performance before I included temperature as a second predictor.  

I also wanted to try alternative methods of predictions to understand how capable (or limited) the LSTM might be in this application. Each model makes a single prediction of the next period, however I used two additional methods to make multiple period predictions by shifting a window of values forward. Confused? Let me illustrate with some simple, visual examples below.  
  
But first, a guide to the colour scheme:  
+ Data enclosed by green borders are predictor sequences  
+ Data enclosed by blue borders predictions to be made  
+ In blue text are predictions that become part of the predictor sequence as the window shifts   

#### Adding dimensions  
The examples below begin by illustrating a scenario where demand was the only input into making predictions. To add some complexity, air temperature will be introduced as an additional dimension and the end of each step. Temperature values have been presented in red text.  
  
#### Data Processing & Prediction Concepts  
Here we have a time-series of demand over 11 periods.    
  
![Seq1](https://github.com/blentley/ForecastingElectricity/blob/master/Screenshots/Seq1.PNG)  
Here it is again with a temperature variable  
  
![Seq6](https://github.com/blentley/ForecastingElectricity/blob/master/Screenshots/Seq6.PNG)  
  
The first shift in our thinking needs to re-frame this from a time-series dataset to a series of sequences. Sequences (or windows as I've sometimes described them) will be used as observations when we train the model.  

If we set a sequence length of 6 periods, then we need to transform this time series into sequence observations as shown below. In this example, I've been able to create six complete sequences of length 6 from these 11 data points.  
  
![Seq2](https://github.com/blentley/ForecastingElectricity/blob/master/Screenshots/Seq2.PNG)  
Here is the example with temperature showing three complete observations of sequence length 6.  
  
![Seq7](https://github.com/blentley/ForecastingElectricity/blob/master/Screenshots/Seq7.PNG)  
  
*How long should a sequence be?*  
It depends on your application. Tune and test your model to find an optimum sequence length.  
  
Once the data is structured in this way, we're ready to start preparing it for training and making predictions. For now, let's continue with understanding different methods of making predictions.  
  
#### Method 1. Predicting the next value, using known data  
In this method of single-step prediction, sequences of known demand data are continually used to predict LP (last prediction).  
  
![Seq3](https://github.com/blentley/ForecastingElectricity/blob/master/Screenshots/Seq3.PNG)  
  
This method of prediction is the most generous since we're providing it with known data all the time to make one prediction at a time. In this method, I'm expecting the error to be the smallest, as the model will likely make slight adjustments from it's known previous value and only be slightly incorrect each time. In practice, there's probably not limited application for a single period forecast, however it's a useful reference in this exercise.  
  
Here is Method 1, with the feature for temperature included for three iterations  
  
![Seq8](https://github.com/blentley/ForecastingElectricity/blob/master/Screenshots/Seq8.PNG)  
  
#### Method 2. Predicting the entire sequence, using only a starting sequence of known data  
In this method of prediction, the row 'Demand 1.1' uses 6 periods of know data to predict a value for time period 7. The next prediction (time period 8) is made using known data from time periods 2 to 6, and the predicted value of time period 7. This process continues until the entirety of the sequence has been predicted. In this method, the known data used for prediction is limited to the initial sequence length, and subsequent predictions are reliant on previously predicted values. By the time we've made it to 'Demand 1.5', only two true values are used for prediction with the remaining values in the sequence coming from P7 to P10.  
  
![Seq4](https://github.com/blentley/ForecastingElectricity/blob/master/Screenshots/Seq4.PNG)  
  
This method of prediction is the least generous (and likely to result in the highest error) since we're only feeding the model limited information and relying on accurate predictions to sustain future predictions. Errors would continue to be amplified as the sequence progresses. However, it still serves as a useful reference point.  
  
Here is Method 2, with the feature for temperature included for three iterations  
  
![Seq9](https://github.com/blentley/ForecastingElectricity/blob/master/Screenshots/Seq9.PNG)  
  
#### Method 3. Predicting multi-period sequences of a defined length  
In this example, let's assume our sequence length is 3 periods and we want to predict a sequence of 2 periods (These can be anything, I've only shortened them to keep the example concise).  

The row 'Demand 1.1' shows that the values in time periods 1 to 3 will be used to predict time period 4. This is a prediction made entirely using known data. Now for the next prediction, 'Demand 1.2', we will use the known values in time periods 2 & 3 and the previously predicted value of time period 4, to predict time period 5. 

This process will reset as we predict 'Demand 2.1', where a fresh set of known values will be used to make a prediction for time period 6.  
  
![Seq5](https://github.com/blentley/ForecastingElectricity/blob/master/Screenshots/Seq5.PNG)  
This method of prediction is somewhat of a middle ground between Methods 1 and 2, where the future predictions are more difficult than simply predicting the next step, but not as difficult as predicting the entire sequence using only a starting seed of known values.  

Here is Method 3, with the feature for temperature included for two iterations  
  
![Seq10](https://github.com/blentley/ForecastingElectricity/blob/master/Screenshots/Seq10.PNG)  
  
For each of these different prediction types, the predicted values can still be compared to their known values during model validation.  

With those concepts confirmed, the next steps will be to prepare the data, build the model, and then make predictions with it.  
  
### Preparing the data  
The first step was to aggregate upwards the 5 minute demand data into an average 30 minute demand so that it would easily join to the air temperature data.  
  
The steps for data preparation have been developed as a series of functions, which I'll run through below. These functions are capable of processing data for univariate and multivariate modelling.

#### 1. Transform raw data into sequences using *prep_data*  
In this function, our input data is transformed into sequences of length *(seq_len)*, before being returned as an array.  
  
```python

def mv_prep_data(inputData, seq_len):

    # Determine the length of the window
    sequence_length = seq_len + 1

    # Create an empty vector for the result
    result = []
    
    for index in range(len(inputData) - sequence_length):
        
        # Append slices of data together
        result.append(inputData[index: index + sequence_length])

    # Convert the result into a numpy array (from a list)
    result = np.array(result)
    
    return result 

```
To use this function, I pass in two parameters:  
+ dataValues - the array of historical demand (and air temperature).  
+ inputSeqLen - the desired sequence length. In this case, I have chosen 1008 periods (21 days).  

```python

inputSeqLen = 1008
dataPrep = mv_prep_data(inputData = dataValues, seq_len = inputSeqLen)

````
The returned result is a series of sequences:  
![DP1](https://github.com/blentley/ForecastingElectricity/blob/master/Screenshots/DP1.PNG)  
  
#### 2. Normalise the data using *normalise_windows*
This function is used to re-scale all the values passed to it relative to the first value in the sequence. The first value in the sequence is scaled to 0 with everything else proportionate to that. This is important especially when additional predictors are added as the LSTM will be sensitve to different scale in input values.  
  
I've included in the return *base_data* are the starting values of each sequence before normalisation. I need this for when it comes time to return the data back to its original scale, I have the reference point to unwind the normalisation.  

```python

def mv_normalise_windows(window_data):
    
    # Create an empty vector for the result
    normalised_data = []

    # Create an empty vector for the values required to undo the normalisation
    base_data = []
    
    # Iteration is over the sequences passed in
    for window in window_data:
        
        # Store the base values
        base_val = window[0]
        base_data.append(base_val)    

        # Perform the normlisation
        normalised_feature = [((window[i] / window[0]) - 1) for i in range(window.shape[0])] 
        normalised_data.append(normalised_feature)

    normalised_data = np.array(normalised_data)
    base_data = np.array(base_data)
    
    return normalised_data, base_data

```
To use function, I pass in the previously sequenced data  

```python

dataNorm, dataBase = mv_normalise_windows(dataPrep)

```
The returned results are normalised sequences and a reference array of base values. You will notice that the base values correspond to first set of values in the previous step.  
![DP2](https://github.com/blentley/ForecastingElectricity/blob/master/Screenshots/DP2.PNG)  
  
#### 3. Partition the data into training and test sets  
This function will take the normalised data and a point to split the dataset, and return six objects:  
+ x_train - an array of sequences used for making predictions.  
+ y_train_y - an array of values that the model will learn to predict.  
+ y_train_x - an array of additional predictors aligned to the newly predicted demand values. Not required in training. 
+ x_test - an array of sequences that will be used for making predictions during validation.  
+ y_test_y - an array of values that we will compare against predictions made by the model during validation. These are the true outcome values.  
+ y_test_x - an array of additional predictors aligned to the newly predicted demand values, which the model needs for subsequent predictions.  
  
In addition, this function will also return *row*, which is the partitioning point of the dataset. Use this to find the correct time-series value for plotting on the x-axis later on.  
  
```python

def mv_split_data(inputData, partitionPoint, outcomeCol):
    
    # Develop a partition to split the dataset into training and test set
    row = round(partitionPoint * inputData.shape[0])
 
    # Create training sets 
    train = inputData[:int(row), :]
    x_train = train[:, :-1]
    y_train = train[:, -1]
    
    y_train_y = y_train[: , outcomeCol]
    # We should also keep the other variable values
    y_train_x = y_train[: , (outcomeCol + 1)]
    
    # Create testing sets
    x_test = inputData[int(row):, :-1]
    y_test = inputData[int(row):, -1]
    y_test_y = y_test[: , outcomeCol]
    y_test_x = y_test[:, (outcomeCol + 1)]
    
    return [x_train, y_train_y, x_test, y_test_y, row, y_train_x, y_test_x]

```
I use this function by passing in the following to the parameters:  
+ dataNorm - the normalised sequences from the previous step.  
+ inputPartition - the % to allocate to model training. Here I've specified 75% of the data to be dedicated for model training.  
+ outcomeCol - the column index which has the variable we're aiming to predict.  

```python

# Partition the data into training and test sets
# Split the data into 75% training and 25% test
inputPartition = 0.75
X_train, y_train, X_test, y_test, rowID, y_train_x, y_test_x = mv_split_data(inputData = dataNorm
                                                                             , partitionPoint = inputPartition
                                                                             , outcomeCol = 0)
```
The returned results are similar in nature to the previous outputs, however they have been split into their respectve parts.  
  
### Assembling the model  
The next step is where we get to assemble the model using Keras.  

Let's first understand what are the inputs:  
+ layers - this is a list of numbers which I specify how many neurons I want in each layer. In my case, I have specified [25, 10, 1], so my first layer will have 25 hours, my second layer will have 10, and my final output layer will have 1 (i.e. my predicted value).  
+ inputTrain - here I pass in X_train, not for training, but to get the dimensions of the data that the model will train on   
  
Now let's break down the components of this function:  
+ Assigning a variable 'model' to Sequential() initiates a sequential model class, which can be thought of as a linear stack of layers.  
+ Next we add layers which for the first one I've specified to be a LSTM layer. In this first layer, we need to specify what shape the data coming in will be, which can be done with the *input_shape* parameter. I then add the second hidden layer which will contain 10 neurons.  
+ I then add the last layer ('Dense'), which will be the output layer with a single neuron.    
+ For simplicity, I've specified the activation function of the neurons to be linear.  
+ Finally, to compile the model, I specify the loss function as the mean-squared error which is what the network will aim to minimise by adjusting its weights during training. The optimisation algorithm used is 'rmsprop', a modified form gradient descent that balances step size which effectively normalises the gradient.  

```python

def build_model(layers, inputTrain):
    
    model = Sequential()
   
    model.add(LSTM(layers[0]
    	, input_shape=(inputTrain.shape[1], inputTrain.shape[2])
    	, return_sequences=True))
    model.add(LSTM(layers[1]))
    
    model.add(Dense(layers[2]))
    model.add(Activation("linear"))

    start = time.time()
    model.compile(loss="mse", optimizer="rmsprop")
    
    print("> Compilation Time : ", time.time() - start)
    
    return model

```  
To use this model, I need to give it some data and other instructions for training:  
+ X_train - this is the prepared sequences of predictors for the model.  
+ y_train - this is the array of true outcome values that the model will try to predict.  
+ batch_size -  this is the size of the data to process in batches within each epoch. Using batches will be more efficient than passing single sequences each time.  
+ inputEpochs - this is the number of training iterations of this neural network, where one iteration has passed all data through the network.  
+ validation_split - this is the proportion of the data to be held out for validation purposes during training.  

```python

fitMSS = modelMSS.fit(X_train, y_train, batch_size = 512, epochs=inputEpochs, validation_split = 0.05)

```
With the model trained, we can now use it for making predictions on unseen test data.

### Using the model to make predictions  
In this step, I've included the functions used to make predictions according to the three methods described above.  
    
The comments in the code provide an overview of the functions key steps. Each of the functions return a simple array of predicted values for comparison to the true values in the test set.  
  
#### Method 1 - predict_point_by_point  
```python

def predict_point_by_point(model, data):
    
    # Use the model to make predictions
    # Return the result in the correct shape  
    predicted = model.predict(data)

    predicted = np.reshape(predicted, (predicted.size,))
    
    return predicted

```
I ran the *predict_point_by_point* function with the following inputs:  
+ X_test - sequences of predictors from the test set  
+ modelMSS -  the model object required to make predictions  

```python

# Make single step predictions with the model
predictMSS = predict_point_by_point(data = X_test, model = modelMSS)

```
  
#### Method 2 - mv_predict_sequence_full    
```python

def mv_predict_sequence_full(model, data, window_size, excess_predictors):
    
    # Begin with starting point
    curr_frame = data[0]
    
    # Create an empty vector to hold predictions
    predicted = []
    
    # Loop over the length of the X_train dataset
    for i in range(len(data)):
        
        # Append the result to the predicted vector        
        predicted.append(model.predict(curr_frame[newaxis,:,:])[0,0])
        
        new_obs = [predicted[-1], excess_predictors[i]]

        # Make space in the predicting frame for the new prediction
        curr_frame = curr_frame[1:]
        
        # Insert the prediction and extra predictors to the end of the frame used for making predictions
        curr_frame = np.insert(curr_frame
                               , [window_size - 1] 
                               , new_obs
                               , axis = 0) 
    
    return predicted

```
I ran the *mv_predict_sequence_full* function using the following inputs:  
+ X_test - sequences of predictors from the test set  
+ modelMSS -  the model object required to make predictions  
+ inputSeqLen - this is your starting sequence length. In my case, I used 1008, which was 21 days (3 weeks)  
+ y_test_x - this is an array of the additional predictors (in this case, temperature) to supplement the predicted demand value as the window is shifted forward  

```python

predictMMS = mv_predict_sequence_full(data = X_test
                                      , model = modelMSS
                                      , window_size = inputSeqLen
                                      , excess_predictors = y_test_x)

```
  
#### Method 3 - mv_predict_sequences_multiple
```python

def mv_predict_sequences_multiple(model, data, window_size, prediction_len, excess_predictors):
    
    # Create an empty vector of predicted sequences
    prediction_seqs = []
    
    # Create a counter for the excess x predictors required
    xs = 0

    # Iterate over multiple chunks of the data
    for i in range(int(len(data) / prediction_len)):
        
        # Create a sequence used for prediction
        curr_frame = data[i * prediction_len]
            
        # Create an array to store predictions made
        predicted = []
        
        # The second loop is to iterate through the prediction length
        for j in range(prediction_len):
            
            predicted.append(model.predict(curr_frame[newaxis,:,:])[0,0])
            
            new_obs = [predicted[-1], excess_predictors[xs]]
            
            curr_frame = curr_frame[1:]
                
            # Append the latest prediction as the last value in the window
            curr_frame = np.insert(curr_frame
                                   , [window_size-1]
                                   , new_obs
                                   , axis=0)
            xs = xs + 1
                            
        prediction_seqs.append(predicted)
        
    return prediction_seqs

```
I ran the *mv_predict_sequences_multiple* function using the following inputs:  
+ X_test - sequences of predictors from the test set  
+ modelMSS -  the model object required to make predictions  
+ inputSeqLen - this is your sequence length. In my case, I used 1008, which was 21 days (3 weeks)  
+ inputPredLen - this is the desired number of predictions to be made. In my case, I specified a prediction length of 48 periods (1 day)  
+ y_test_x - this is an array of the additional predictors (in this case, temperature) to supplement the predicted demand value as the window is shifted forward  

```python

predictMMS = mv_predict_sequences_multiple(data = X_test
                                        , model = modelMSS
                                        , window_size = inputSeqLen
                                        , prediction_len = inputPredLen
                                        , excess_predictors = y_test_x)

```
  
### Modelled outcomes  
One last, but important set of functions take the predicted values and revert them back to their original scale. I could have left the results normalised, but I felt the values lose their context that the true scale provides.  

This was done with the *mv_return_original_scale* and *return_original_scale_multiple* functions for Method 1 and Method 3 respectively. The reversal of the normalisation for Method 2 didn't not require a function as referencing the first data point as simple enough.  

#### Method 1- Reversing normalisation with *mv_return_original_scale*  
```python

def mv_return_original_scale(norm_val, base_val):
    
    rescaled = (norm_val + 1) * base_val
    
    return rescaled

```
I ran the *mv_return_original_scale* function with the following inputs:  
+ predictMSS - an array of normalised predicted values from the single step model  
+ referenceVal - the normalisation reference values required to un-normalise the predictions  

```python

predictMSS_scaled = mv_return_original_scale(norm_val = predictMSS, base_val = referenceVal)

```

#### Method 3 - Reversing normalisation with *return_original_scale_multiple*  
```python

def return_original_scale_multiple(norm_val, base_val, prediction_len):
    
    # Create an empty an empty array
    seq_base = []

    # First loop over number of sequences
    for i in range(int(len(base_val) / prediction_len)):
        
        # Second loop through the number of predictions
        # Repeat the reference value
        for j in range(prediction_len):
            seq_base.append(base_val[i * prediction_len])
            
    seq_base = np.array(seq_base)
            
    # Reshape the normalised predictions
    # Reduce the dimensionality of the dataset, back into a single array
    newRowDim = norm_val.shape[0] * norm_val.shape[1]
    norm_val_reshaped = norm_val.reshape(newRowDim)

    # Cross multiply the two arrays to rescale the predictions
    rescaled = seq_base * (norm_val_reshaped + 1)
    
    return rescaled, newRowDim

```
I ran the *return_original_scale_multiple* function with the following input:  
+ predictMMS - the normalised predictions from the multi period model  
+ referenceVal - the normalisation reference values required to un-normalise the predictions  
+ inputPredLen - the length of time I specified predictions for (in this case it was 48 periods)  

```python

predictMMS_scaled, newRowDim = return_original_scale_multiple(norm_val = predictMMS
                                                              , base_val = referenceVal
                                                              , prediction_len = inputPredLen)

```
  
The results presented below are plots showing the actual values of demand and the predicted values of demand for the test set. I've also included a subsection of these results for clearer viewing.  

#### Method 1 results  
As expected the predicted values follow quite closely to the observed values. The root mean squared error (RMSE) I've used for evaluation is 279, the lowest of the three methods.  
**Full results**  
  
![M1Full](https://github.com/blentley/ForecastingElectricity/blob/master/Screenshots/M1FullResults.PNG)  
**Partial results**  
  
![M1Section](https://github.com/blentley/ForecastingElectricity/blob/master/Screenshots/M1SectionResults.PNG)  

#### Method 2 results  
This method was always going to be the most difficult of the prediction methods, and it is reflected in the RMSE of 1647. The plot shows that the prediction doesn't follow the trends of the observed values closely at all, and converges into a narrow range with some noise.  
**Full results**  
  
![M2Full](https://github.com/blentley/ForecastingElectricity/blob/master/Screenshots/M2FullResults.PNG)  

**Partial results**  
  
![M2Section](https://github.com/blentley/ForecastingElectricity/blob/master/Screenshots/M2SectionResults.PNG)  

#### Method 3 results  
This method was anticipated to be the middle ground of Methods 1 and 2, and it was confirmed by the RMSE of 1057. Given the task of predicting the next 48 periods (24 hours), looking at the plot of full results it seems to be able to follow the trend, but consistently underpredicts the values.    
**Full results**  
  
![M3Full](https://github.com/blentley/ForecastingElectricity/blob/master/Screenshots/M3FullResults.PNG)  

**Partial results**  
  
![M3Section](https://github.com/blentley/ForecastingElectricity/blob/master/Screenshots/M3SectionResults.PNG)  

#### The impact including temperature as a predictor  
The results presented above were for models developed with both historical demand and air temperature used as predictors. The understand the impact of including temperature as a predictor of electricity demand, I have summarised the RMSE for modelled scenarios across the three scenarios in the table below.  
  
![Results](https://github.com/blentley/ForecastingElectricity/blob/master/Screenshots/Results.PNG)  

The results indicate the addition of temperature as a predictor has a positive impact on predictive capability across all three methods.  

The script where I performed the LSTM modelling can be found in Scripts/PredictingDemand.ipynb. There is also an equivalent HTML output.  

## Final Thoughts
I have presented my very preliminary exploration of using LSTMs to predict electricity demand. I'll leave you with some ideas I had on how this could be improved (there are probably many more, no doubt):  
+ Try to optimise the structure of the LSTM - The model I've specified above is an arbitrary combination of hidden layers, neurons and epochs, but could be optimised through a parameter search at scale.  
+ Use the LSTM to natively predict multiple periods - in this guide, I have specified one output from the LSTM before using methods of shifiting windows forward to make longer predictions. I'd be interested to see what the LSTM would return if it is told to return an x period prediction.  
+ Streamlining functions - you'll see in the notebooks attached, I have used separate functions for my univariate and multivariate modelling. The multivariate functions should be applicable for scenarios of all dimensions.  
+ Try different pre-processing of the data - I've chosen an arbitrary sequence length of 21 days, but this is not optimised. [Research](http://www.aemc.gov.au/getattachment/924537dd-1f48-4550-a134-78b3b7d3ba70/University-of-Wollongong,-Evaluation-of-Neural-Net.aspx) done by the University of Wollongong suggests a longer time period capturing seasonality could be significant.  
+ Include a longer period of training by including more historical data from 2015 and earlier.  
+ Try additional predictors - This could take the form of additional weather stations, additional climate features (humidity, rainfall) or smart meter data.  
